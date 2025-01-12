---
title: GPU timing query in Metal API
description: How to implement writeTimestamp for Metal API in multi-api engine.
author: snowapril
date: 2025-01-12 20:00:00 +0800
categories: [Metal, Dev]
tags: [metal]
pin: true
math: true
mermaid: true
---

<!-- markdownlint-capture -->
<!-- markdownlint-disable -->
# How to implement writeTimestamp for Metal API in multi-api engine.
{: .mt-4 .mb-0 }
<!-- markdownlint-restore -->
Write timestamp at any time is not supported in TBDR architecture.
In metal api, it support only querying timing on stage boundary for TBDR(a.k.a. apple silicon gpu).

For the big engine that supports multiple graphics API, it is quite painful that graphics api does not support query timing at any time (of course it is quite different according to hardware and not really any time internally). So we need to decide between not supporting timing query at any time or taking charge of performance regression for calculate and organize helpful timing informations.

While browsing many open-source engine’s decision, I think it might be easy to implement with little code stuff and some performance overhead.

1. Everytime before MTLCommandEncoder create, attach sample buffer attachment at final stage for each type of encoder.
2. For each writeTimestamp query request from engine, split current opened MTLCommandEncoder. This may cause performance overhead according to MTLCommandEncoder type and number of attachments with store action if encoder is MTLRenderCommandEncoder. Also Metal API doing synchronize each batch with MTLFences if they exist in difference encoders instead of memory barrier which is quite chip than fence.

With this scenario, we can implement writeTimestamp quite easier. But if there are many drawcalls in MTLRenderCommandEncoder, which drawcall’s fragment stage end time will be recorded in that sample buffer attachment? If only first drawcall’s fragmnet stage end time is recorded, we can not apply this scenario. Unfortunately I cannot find the answer for this problem in the Metal API spec. 
So I tried this in my M3 Pro TBDR laptop with below codes.

In DeviceSelectionAndFallback metal sample code(no special reason select it), I add MTLCounterSampleBuffer creation and add MTLRenderPassSampleBufferAttachmentDescriptor to MTLRenderPassDescriptor every frame and print timestamp value for every completion callback for that frame.

```objective-c
MTLRenderPassSampleBufferAttachmentDescriptorArray* renderPassDescArr = [renderPassDescriptor sampleBufferAttachments];
    [[renderPassDescArr objectAtIndexedSubscript:0] setSampleBuffer:_sampleBuffer];
    [[renderPassDescArr objectAtIndexedSubscript:0] setStartOfVertexSampleIndex:MTLCounterDontSample];
    [[renderPassDescArr objectAtIndexedSubscript:0] setEndOfVertexSampleIndex:MTLCounterDontSample];
    [[renderPassDescArr objectAtIndexedSubscript:0] setStartOfFragmentSampleIndex:MTLCounterDontSample];
    [[renderPassDescArr objectAtIndexedSubscript:0] setEndOfFragmentSampleIndex:frameNumber * 3];
    
    [[renderPassDescArr objectAtIndexedSubscript:1] setSampleBuffer:_sampleBuffer];
    [[renderPassDescArr objectAtIndexedSubscript:1] setStartOfVertexSampleIndex:MTLCounterDontSample];
    [[renderPassDescArr objectAtIndexedSubscript:1] setEndOfVertexSampleIndex:MTLCounterDontSample];
    [[renderPassDescArr objectAtIndexedSubscript:1] setStartOfFragmentSampleIndex:MTLCounterDontSample];
    [[renderPassDescArr objectAtIndexedSubscript:1] setEndOfFragmentSampleIndex:frameNumber * 3 + 1];
    
    [[renderPassDescArr objectAtIndexedSubscript:2] setSampleBuffer:_sampleBuffer];
    [[renderPassDescArr objectAtIndexedSubscript:2] setStartOfVertexSampleIndex:MTLCounterDontSample];
    [[renderPassDescArr objectAtIndexedSubscript:2] setEndOfVertexSampleIndex:MTLCounterDontSample];
    [[renderPassDescArr objectAtIndexedSubscript:2] setStartOfFragmentSampleIndex:MTLCounterDontSample];
    [[renderPassDescArr objectAtIndexedSubscript:2] setEndOfFragmentSampleIndex:frameNumber * 3 + 2];

// begin render command encoder
// call three draw call 
// end render command encoder
// commit

// At command completion callback
[commandBuffer addCompletedHandler:^(id<MTLCommandBuffer> buffer)
     {
        NSData* resolvedData = [_sampleBuffer resolveCounterRange:NSMakeRange(frameNumber * 3, 3)];
        MTLCounterResultTimestamp* timestamps = (MTLCounterResultTimestamp *)(resolvedData.bytes);
        NSLog(@"0 : %llu, 1 : %llu, 2 : %llu", timestamps[0].timestamp, timestamps[1].timestamp, timestamps[2].timestamp);
     }];
```

I thought it will print every unique timestamp values for every frame, but it’s wrong as shown below.

```text
0 : 0, 1 : 0, 2 : 5128184060000
0 : 0, 1 : 0, 2 : 5128194340208
0 : 0, 1 : 0, 2 : 5128210927041
0 : 0, 1 : 0, 2 : 5128226770833
0 : 0, 1 : 0, 2 : 5128243420833
0 : 0, 1 : 0, 2 : 5128257669083
0 : 0, 1 : 0, 2 : 5128273305416
0 : 0, 1 : 0, 2 : 5128289183041
0 : 0, 1 : 0, 2 : 5128305873541
0 : 0, 1 : 0, 2 : 5128322426333
...
```

According to experiment result, it seems record only for last draw call’s endOfFragmentStage timestamp on sampleBuffer. There was no difference in the timestamp print results whenever I add MTLRenderPassSampleBufferAttachmentDescriptor.

With this result, we can implement writeTimestamp query with predefined scenarios.
1. Every time MTLCommandEncoder creation, attach sampleBuffer at the last stage of that command encoder type.
2. Whenever writeTimestamp requested, split command encoder and assign the index of sampleBuffer to that request which used for that command encoder creation.
3. At the command buffer completion callback or any time command completion is ensured, resolve.

But we must notice that command encoder splitting has quite big overhead according to command encoder’s type and number of store action invocations.
