---
layout: post
title: A Feature Toggle Story
tags: [feature toggle]
---

In the following article, I'll talk about how the feature toggle technique helped me to introduce a new feature even when not all dependencies were ready. How the same technique enabled the delivery of a big refactor. Finally, another use case I will touch on is how was possible to altered the behavior of a system based on configurable feature toggles and user info.

### Delivering a new feature with dependencies
Not so long ago I had to modify the flow of the PDF generator service in order to integrate the PDF signing feature. The main class looked like following:

```java
@AllArgsConstructor
@Service
class ReportService {
    PDFGenerator generator;
    StorageService storageService;

    void generateReport(Payload payload) {
        OutputStream reportStream = generator.generate(payload);
        storageService.upload(reportStream);
    }
}
```
The tricky part of this task was the fact that I didn't get a certificate straight away. The discussion about certificate provider and other specifics started to get tedious, and I didn't have the certainty that it will be ready in a few hours or weeks.

During analysis I find out that the PDF generation is an important step and could not suffer any disruptions. An aspect that I took into consideration during the design phase. 

For the signing step I created an interface:

```java
interface SignerService {
    OutputStream sign(OutputStream payloadToSign);
}
```

And integrated it into the main logic:

```java
@AllArgsConstructor
@Service
class ReportService {
    PDFGenerator generator;
    SignerService signerService;
    StorageService storageService;

    void generateReport(Payload payload) {
        OutputStream reportStream = generator.generate(payload);
        OutputStream signedReportStream = signerService.sign(reportStream);
        storageService.upload(signedReportStream);
    }
}
```

After this I wanted to ensure that the service is working as before and created a _noop_ implementation:

```java
@Service
class NoOpSignerService implements SignerService {

    @Override
    public OutputService sign(OutputService payloadToSign) {
        log.info("bypassing signing step");
        return payloadToSign;
    }
}
```

Finally, it was the time to get to the problem at hand and actually sign the report. For this part I defined another implementation, that is `@Primary` and loaded only in certain conditions:

```java
@Service
@Primary
@ConditionalOnProperty(name = "activateSigning", havingValue = "true")
class BCSignerService implements SignerService {

    @Override
    public OutputService sign(OutputService payloadToSign) {
        //... sign and return the payload
    }
}
```
This means that we have two implementations, but the second one which takes precedence over any other beans is loaded into the Spring context only when `activateSigning` is `true`.

It seems like a trivial thing to do, but thanks to this class structure I was able to develop, merge into the main branch (so the dev brach does not get stale) and even demo to the client (with a self signed certificate).
When the certificate was finally available, I just switched the flag `activateSigning` to `true`, did a restart of the service, and everything worked perfectly (preferably to restart is blue-green deployment so you don't have any downtime).

About how I used the feature toggle for other goals mentioned in the beginning, details will follow soon.