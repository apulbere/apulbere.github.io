---
layout: post
title: A Feature Toggle Story
tags: [feature toggle]
---

Feature toggle is a technique that helps to change the system's behavior without altering the code. It's interesting how stories around it vary from the catastrophic failure of the Knight Capital Group<sup>[1]</sup> to the stellar success of the first GPS satellite<sup>[2]</sup>.
In the following article, we'll see a more typical example that all engineers have to deal with.

### Delivering a new feature according to Murphy's Law
Imagine an existing flow in a system that generates a PDF report, and then uploads it to the cloud:

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
Now the client wants to add a layer of authentication and sign the PDF before uploading. So let's define an interface just for that:

```java
interface SignerService {
    OutputStream sign(OutputStream payloadToSign);
}
```

And integrate it into the main flow:

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
Everything seems perfectly fine until uncertainties start to creep in.
Let's say we already merged into the main branch the change and even did a demo to the client using a self-signed certificate on the local environment. But shortly before deploying the new version of the system on upper environments, the DevOps informs us that they have problems installing a valid certificate.

What do we do now? The main flow will certainly be broken. Not only the client won't be able to receive his signed PDF, but he won't receive a PDF at all. So, nothing left but to frantically revert the change, fix some conflicts on the way, run the regression, and finally prepare a new version. 

Now let's imagine another scenario where right from the beginning we took into consideration Murphy's law. In this scenario, even after we implemented the feature, the main flow should work as before and only when we wish so, to pass through the signing step.

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
Above, is a noop implementation for the code to work as before. And below is another implementation that does the signing.

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
Notice that this implementation is the primary one but Spring picks it up and injects it into the main flow only when `activateSigning` is `true`.

### Static vs dynamic feature toggles
Perhaps one of the objections to the above solution would be the fact that we have to change an environment variable (`activateSigning`) and do a restart.
True, but this doesn't necessarily mean that your service will be offline since you can have multiple instances and gradually replace them.

This approach is a _static_ one, meaning that your application will work only with one state of the feature toggle, flipping it will not change the running system. There are certain use cases and reasons to choose the static feature toggle:
1. It's one time only - you won't need to go back and forth between states of the flag unless you have to "rollback" and revise it.
2. The code is clean, testable, and modular. Notice that the feature lives in a separate class, apart from the rest of the code. The old unchanged tests still work, we just have to add a couple of test cases for the activated feature.

The _dynamic_ feature toggles are more fit for scenarios where they are dependent on user requests or the stakeholders want to control them via some management tool.

All things said, I think the feature toggle should be always considered during the design phase even if the client didn't specifically request it and even if we don't have a feature toggle discipline. We can always start small, with configuration properties as shown in this article.

<sup>[1]</sup>[How to lose half a billion dollars with bad feature flags](https://blog.statsig.com/how-to-lose-half-a-billion-dollars-with-bad-feature-flags-ccebb26adeec)

<sup>[2]</sup>[The Relativity Switch](https://www.artsjournal.com/artfulmanager/main/the-relativity-switch.php)

<sup>[3]</sup>[Murphy's law](https://en.wikipedia.org/wiki/Murphy%27s_law)
