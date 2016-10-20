---
layout: post
title: First Impressions of Complex Event Processing in Apache Flink
---

I've been using [Apache Flink](https://flink.apache.org/) to perform streaming aggregations for real-time alerting across several attributes. It's a phenomal stream-processing engine with an easy-to-use API, simple switching between local and cluster job execution, and great support for job checkpointing to simplify recovery when systems fail.

[Flink includes support](https://ci.apache.org/projects/flink/flink-docs-release-1.1/apis/streaming/libs/cep.html) for [Complex Event Processing (CEP)](https://en.wikipedia.org/wiki/Complex_event_processing). CEP allows for pattern-matching against streaming events. A CEP pattern can contain multiple conditions that may be sequential or non-sequential (i.e. non-matching events can exist between), as indicated by the programmer. Pattern conditions can also have time restrictions, causing the pattern match to timeout if conditions are not satisfied in the time constraints given.

I've wanted to use CEP to handle state transitions for alerts triggered in our system. The sequence might look like:

1. Define alerting pattern with one condition that looks for stream elements that meet alerting criteria (i.e. error-rate above threshold)
1. Create CEP pattern with event stream
  1. Select for matches on CEP pattern
    1. Send alert message to external system responsible for managing alerts, sending customer notifications, etc
1. Define alert resolution pattern with one condition that looks for stream elements in a non-alerting state (i.e. error-rate below threshold) within some time (e.g. 24 hours)
1. Create CEP pattern with the *output stream from CEP alerting pattern* and the alert resolution pattern
  1. Select for matches on CEP pattern
    1. Send alert message to external system to clear the alert
  1. Select for *timeout* on CEP pattern
    1. Send alert message to external system to mark alert as auto-resolved

The events that feed into the first alerting CEP pattern are the result of window & transformation operations performed upstream; the events include the error-rate observed over a counting-window in various dimensions. I noticed that the transformed events were not triggering the CEP pattern conditions as I'd expected. No errors were reported either.

The CEP pattern for triggering alerts looks like:

```java
final Pattern<ErrorRateResult, ?> alertPattern = Pattern.<ErrorRateResult> begin("alert")
		.where(evt -> evt.getErrorRate() >= ALERT_THRESHOLD);

final PatternStream<ErrorRateResult> pattern = CEP.pattern(errorRateResultStream, alertPattern);
final DataStream<ErrorRateResult> alertStream = pattern
		.select(new PatternSelectFunction<ErrorRateResult, ErrorRateResult>() {
			private static final long serialVersionUID = 1L;

			@Override
			public ErrorRateResult select(Map<String, ErrorRateResult> events) throws Exception {
				// send alert
				final ErrorRateResult event = events.get("alert");
				return event;
			}
		});
alertStream.print();
```

I was certain there were events in the `errorRateResultStream` data-stream that had a high error-rate, but nothing was being printed. I was running Flink locally in Eclipse, which made it trivial to attach a Java debugger and step through the CEP event processing.

The events were being received by the CEP pattern, which was reassuring. The problem was that the `AbstractCEPPatternOperator` and `AbstractKeyedCEPPatternOperator` have `processWatermark` methods that compare the timestamp on the event with the stream watermark, which would work fine if only my stream had event timestamps still! Since the timestamp on the event was greater (significantly so) than the watermark, it was enqueued for later processing, which would never have happened. The machine would run out of memory before the events were processed.

The transformations performed before the CEP pattern resulted in the event stream timestamps being stripped. Since the execution environment relied on event-time processing, the CEP pattern insisted upon using event-time timestamps.

I corrected this by adding a call to `assignTimestampsAndWatermarks` on the data stream supplied to the CEP pattern:

```java
errorRateResultStream = errorRateResultStream.assignTimestampsAndWatermarks(new AssignerWithPunctuatedWatermarks<ErrorRateResult>() {

	@Override
	public long extractTimestamp(ErrorRateResult element, long previousElementTimestamp) {
		return element.getEndTimestamp();
	}

	@Override
	public Watermark checkAndGetNextWatermark(ErrorRateResult lastElement, long extractedTimestamp) {
		return new Watermark(lastElement.getEndTimestamp());
	}
	})
```

Problem solved!

I look forward to doing more work with Flink's CEP library. It's got a very nice syntax, and the event chaining and time restrictions are exactly what I've been looking for.
