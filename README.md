# OPI Android SDK

A Kotlin SDK for integrating OPI-compatible payment terminals with Android POS and ECR applications over a network connection to a separate terminal.

## Installation

Download the `.aar` from the [latest release](https://github.com/richiehug/opi-android-sdk/releases/latest) and [add it to your application](integration.md#add-the-sdk-to-an-application).

## Requirements

- Android 8.0 (API 26)+
- Java 17 toolchain

## Quick start

```kotlin
val opiSdk = OpiAndroidSdk.create(
    context,
    OpiSdkOptions(terminalHost = "192.168.0.69")
)

suspend fun takePayment() {
    val result = opiSdk.payment(
        amount = 1250,
        currency = "CHF"
    )

    when (result.status) {
        ResultState.SUCCESS -> showSuccess(result)
        ResultState.DECLINED -> showDecline(result.errorCode)
        ResultState.UNKNOWN -> showReconciliationRequired(result)
        else -> showPaymentError(result)
    }
}
```

`SUCCESS` confirms the financial outcome. Persist the available receipts separately. `UNKNOWN` requires reconciliation before retrying. See [network interruption and receipt recovery](integration.md#network-interruption-and-receipt-recovery).

Amounts use the currency's minor unit, so `1250` represents CHF 12.50. Configure the terminal network address explicitly. Applications own their transaction experience and observe the SDK's events and results.

## What the SDK provides

- Take payments, issue refunds, and reverse transactions through a clean Kotlin API.
- Run terminal operations and retrieve terminal information directly from your app.
- Handle every outcome with typed results and structured errors.
- Print merchant and customer receipts or return them to your app for display and storage.
- Offer Dynamic Currency Conversion (DCC) and receive the complete result details.
- Use the payment experience in English, German, French, or Italian.
- Configure timeouts, receipt handling, and Force Acceptance to match your integration.
- Diagnose integration issues with optional, redacted file logging.

## Documentation

- [Integration guide](integration.md)
- [Kotlin SDK API reference](https://richiehug.github.io/opi-android-sdk/)

The API reference documents all public methods, request and response models, result states, errors, configuration options, and events.

## Requirements

- Android 8.0 / API 26 or newer
- Kotlin application with coroutine support
- Java 17 to build the project
- A provisioned payment application exposing compatible OPI TCP endpoints, on a separate network terminal

## Support

For help, questions, or bug reports, [open an issue on GitHub](https://github.com/richiehug/opi-android-sdk/issues).

## License

The compiled SDK is available under the [OPI Android SDK Binary License](LICENSE), including commercial application integration and bundling of the unmodified SDK. Source code remains private. This is not an open-source SDK license.

OPI Android SDK is an independent software project created and maintained by [Richard Hug](https://richiehug.com). This SDK is not owned, maintained, supported, warranted or endorsed by payment providers. It follows the OPI protocol specifications.

Company references describe compatibility and tested environments only. There is no SLA, guaranteed response or resolution time, release schedule, or commitment to resolve individual issues. Its goal is simple: make the OPI protocol brilliantly straightforward to use from Android.
