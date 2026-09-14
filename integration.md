# Integrating the OPI Android SDK

This guide covers installation, configuration, terminal operations, result handling, recovery, and diagnostics. For an SDK overview and platform requirements, see the [README](README.md).

## Add the SDK to an application

### Local AAR

Download `opi-android-sdk-1.0.0.aar` from the [GitHub release](https://github.com/richiehug/opi-android-sdk/releases/tag/1.0.0) and copy it to the application's `app/libs` directory.

Reference the local AAR and its runtime dependencies from the application module:

```kotlin
dependencies {
    implementation(files("libs/opi-android-sdk-1.0.0.aar"))
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.9.0")
}
```

The local AAR does not carry Maven dependency metadata, which is why the coroutine runtime dependency must be declared explicitly.

## Initialize the SDK

Create one SDK instance and retain it at activity, ViewModel, dependency-injection, or application scope:

```kotlin
val opiSdk = OpiAndroidSdk.create(context, OpiSdkOptions(terminalHost = "192.168.0.69"))
```

Configure the language, receipt handling, and terminal address when needed:

```kotlin
val opiSdk = OpiAndroidSdk.create(
    context,
    OpiSdkOptions(
        language = "de",
        receiptHandling = ReceiptHandling(
            merchantReceipt = ReceiptHandlingMode.Available,
            customerReceipt = ReceiptHandlingMode.Available
        ),
        terminalHost = "192.168.0.69"
    )
)
```

Supported language codes are `en`, `de`, `fr`, and `it`. Full configuration details and defaults are available in the `OpiSdkOptions` KDoc linked above.

Import `ReceiptHandling` and `ReceiptHandlingMode` from `io.github.richiehug.opi.android.sdk.model`.

## Network integration

Configure the address of a separate OPI terminal explicitly. The terminal must be able to connect back to the Android device on the configured device-channel port (4102 by default); the outbound pay channel defaults to 4100. Coordinate firewall rules and routing for both channels.

The SDK declares INTERNET, ACCESS_NETWORK_STATE and WAKE_LOCK. These normal permissions support TCP communication, network-loss detection and bounded operation wake locks. No service registration or runtime permission prompt is required. Retain the SDK and operation coroutine in an application-owned lifecycle scope until the operation finishes.

Use `operationEventListener` for Started, Connected, ResultUncertain, Completed, receipt, terminal-message, DCC notifications. Call `abort()` for an explicit abort request. A successful abort request is not the final payment outcome: continue awaiting the original operation result. Listeners may run on background threads and must return promptly. The demo APK contains a complete application integration driven by these events.

## Receipt handling

The final ep2 merchant and customer receipts can be configured independently:

```kotlin
val options = OpiSdkOptions(
    receiptHandling = ReceiptHandling(
        merchantReceipt = ReceiptHandlingMode.Available,
        customerReceipt = ReceiptHandlingMode.Available
    )
)
```

| SDK value | Behaviour |
| --- | --- |
| `PrintLocal` | The terminal prints the ticket. No ticket text is expected in the SDK result. |
| `Available` | The terminal returns the printable ticket in the SDK result. |

**Recommended:** explicitly configure both copies as `Available`, as in the example above. This works for printerless terminals and lets the ECR store receipts before printing. `PrintLocal` requires a working terminal printer; requesting it on a printerless terminal can cause `DeviceUnavailable` and leave a pending receipt. Both copies default to `Available`; select `PrintLocal` explicitly only when the terminal should print them. The SDK returns them through the public `customerReceipt` and `merchantReceipt` fields. It does not expose a journal receipt type. The terminal's E-Journal remains available independently.

Returned copies are read directly from the operation result:

```kotlin
val merchantTicket = result.transaction?.receipts?.merchantReceipt?.content
val customerTicket = result.transaction?.receipts?.customerReceipt?.content
```

Receipt content is plain text with line breaks and alignment spaces retained. Each operation exposes only its final ep2 merchant and customer tickets. No additional journal type is exposed; DCC offers and intermediate printer messages are acknowledged internally and are not part of the SDK result.

Display tickets using a monospace font. When receipt handling is `Available`, the integrating application is responsible for printing or securely persisting the returned tickets. Persist the content together with the transaction before discarding the result. A missing receipt is normal when its mode is `PrintLocal`, or when no final ticket of that type was generated.

## Payments

Amounts use the currency's minor unit. For example, CHF 12.50 is passed as `1250`.

```kotlin
val result = opiSdk.payment(
    amount = 1250,
    currency = "CHF"
)
```

The amount must be positive and the currency must be a valid three-letter ISO 4217 code. An optional merchant reference can be passed in `reference` (limited to 20 characters).

## Refunds

A refund can be started with amount and currency:

```kotlin
val result = opiSdk.refund(
    amount = 1250,
    currency = "CHF"
)
```

When the terminal workflow requires a referenced refund, provide the `authReference` returned by the original payment:

```kotlin
val result = opiSdk.refund(
    amount = 1250,
    currency = "CHF",
    authReference = "123456"
)
```

## Reversals

```kotlin
val result = opiSdk.reversal()
```

On Swiss ep2 terminals this is **Reverse Last**: it can cancel only the last transaction. No amount, currency, approval code, or transaction reference is supplied by the SDK. The merchant application must restrict this action to an authorized operator. Use `refund(...)` when money must be returned for a specific or older transaction.

## Terminal operations

| SDK method | Purpose |
| --- | --- |
| `abort()` | Request cancellation of an active card operation |
| `activate()` | Activate the terminal |
| `close()` | Perform the terminal close-day operation |
| `config()` | Request terminal configuration synchronization |
| `deactivate()` | Deactivate the terminal |
| `info()` | Read terminal and application information |
| `init()` | Request acquirer initialization |
| `login()` | Start the login flow |
| `logoff()` | End the login session |
| `reprint()` | Reprint the terminal's last ticket |
| `reset()` | Restart the terminal |
| `status()` | Read the terminal status and available terminal information |
| `submit()` | Submit and transmit pending transactions |

All methods are suspending:

```kotlin
suspend fun refreshTerminalInfo() {
    val result = opiSdk.status()
    showTerminalInfo(result)
}
```

Only one regular operation can run at a time; overlapping calls return `PENDING_TRANSACTION`. `abort()` is the out-of-band operation that can interrupt an active payment or refund.

## Handle results

Every operation returns `OpiResult`. Lifecycle integrations can set `OpiSdkOptions.operationEventListener` to receive structured start/completion, terminal-message, receipt, DCC, and connection-status events. Listener failures are isolated from the operation result.

Depending on the operation and terminal response, a successful result can contain:

- Merchant reference, authorization reference/code, transaction date, and acquirer ID
- Final amount, tip, currency, standardized payment method, and masked PAN in `transaction`
- Merchant and customer printable receipt text
- DCC offer/acceptance state, cardholder amount and currency, numeric currency code, exchange rate, and markup
- Terminal identity, status, version, and configuration
- Classified `status`, a typed `error`, a detailed `errorCode` when available, and a UUID `correlationId`


`transaction.reference` is the optional merchant reference supplied with the payment. The terminal-generated authorization reference is returned separately as `transaction.authReference` and is the value to use for a referenced refund.

### Payment method

The SDK returns a stable lowercase payment method for known brands:

```kotlin
val paymentMethod = result.transaction?.paymentMethod // for example "visa", "mastercard", or "twint"
val maskedPan = result.transaction?.maskedCardNumber
```

Supported normalized values include `visa`, `mastercard`, `maestro`, `vpay`, `amex`, `jcb`, `diners`, `discover`, `unionpay`, `girocard`, and `twint`. Unrecognized values are returned in lowercase for forward compatibility.

### Tip results

The final amount and optional tip are available on the payment result:

```kotlin
val finalAmount = result.transaction?.amount // minor units, includes tip
val tip = result.transaction?.tip             // minor units, e.g. 2 for CHF 0.02
```

`tip` is `null` when no positive tip was added. Both values use minor currency units, consistent with the amount passed to `payment()`. When SDK logging is enabled, the structured result summary includes the final amount and tip; raw terminal communication remains redacted as described below.

### DCC results

`result.transaction?.dcc` is populated only when the terminal offered DCC and returned complete DCC details. Other transactions return `null`.

```kotlin
result.transaction?.dcc?.let { dcc ->
    check(dcc.offered)
    if (dcc.accepted) {
        showDcc(
            amount = dcc.amount,
            currency = dcc.currency,
            currencyCode = dcc.currencyCode,
            exchangeRate = dcc.exchangeRate,
            markupPercentage = dcc.markupPercentage
        )
    }
}
```

`currencyCode` contains the terminal's numeric ISO code when supplied; `currency` contains the alphabetic code.

### Errors and cancellations

A response is always classified through `status`:

```kotlin
when (result.status) {
    ResultState.SUCCESS -> showSuccess(result)
    ResultState.DECLINED -> showDecline(result.errorCode)
    ResultState.ABORTED -> showCancelled()
    ResultState.COMMUNICATION_ERROR -> showConnectionError(result.errorCode)
    ResultState.TERMINAL_ERROR -> showTerminalError(result.errorCode)
    ResultState.INVALID_REQUEST -> showValidationError(result.errorCode)
    ResultState.IN_PROGRESS -> showOperationInProgress()
    ResultState.UNKNOWN -> showUnknownResult()
}
```

In particular:

- `ABORTED` is a completed cancellation, not a transport error.
- `DECLINED` is derived from host or terminal decline evidence. The available decline reason is returned in `errorCode`.
- `INVALID_REQUEST` includes terminal `ParsingError` and schema or data rejection.
- `COMMUNICATION_ERROR` with `error = CONNECTION_ERROR` means the SDK could not connect to the payment application.
- `UNKNOWN` with `error = RESULT_UNKNOWN` means the request was sent but no reliable final result arrived; reconcile before retrying.

Use the typed `result.error` for stable application logic. `errorCode` carries the more specific terminal or SDK reason when one is available.

| `OpiError` | Example `errorCode` values | Meaning and expected handling |
| --- | --- | --- |
| `FAILURE` | Terminal reason such as `5` or `569`, otherwise absent | The terminal returned a generic failure. Inspect `status` to distinguish a decline from another terminal failure. |
| `ABORTED` | `ABORTED` | The cashier, cardholder, or SDK abort flow cancelled the operation. |
| `REPRINT_REQUIRED` | `PRINTLASTTICKET` | The terminal has blocked new transactions until the previous ticket is reprinted and acknowledged. |
| `DEVICE_UNAVAILABLE` | `DEVICEUNAVAILABLE` | The payment application is temporarily unavailable or is still presenting the previous result. Wait before retrying. |
| `TERMINAL_BUSY` | `BUSY` | An operation is still in progress. Do not start a competing financial request. |
| `LOGIN_REQUIRED` | `LOGGEDOUT` | Activate the terminal before retrying the operation. |
| `RESULT_UNKNOWN` | `RESULT_UNKNOWN` | The request was sent but no reliable final response arrived. Reconcile before retrying; never assume failure. |
| `CONNECTION_ERROR` | `CONNECTION_ERROR` | The SDK could not establish the terminal connection, so the request was not sent. |
| `OPERATION_TIMEOUT` | `OPERATION_TIMEOUT`, `TIMEDOUT`, `FLOWTIMEDOUT` | The SDK or terminal flow exceeded its timeout. Check `status`; reconcile if delivery might have occurred. |
| `INVALID_REQUEST` | `INVALID_REQUEST`, `FORMATERROR`, `PARSINGERROR`, `VALIDATIONERROR`, `MISSINGMANDATORYDATA` | SDK validation or terminal XML/schema validation rejected the request. Correct the request before retrying. |
| `OTHER` | Any unrecognized terminal result | Preserve the error code for diagnostics and avoid guessing its meaning. |

For example, a terminal failure with reason `569` returns `status = TERMINAL_ERROR`, `error = FAILURE`, and `errorCode = "569"`. A host decline returns `status = DECLINED`, `error = FAILURE`, and the available decline reason in `errorCode`.

#### Timeouts and lost connectivity

The SDK has four relevant timeout controls. Their defaults are:

- `connectTimeoutMillis = 10_000`: maximum time allowed to establish the terminal connection. If no request was sent, the result is `CONNECTION_ERROR`.
- `operationTimeoutMillis = 300_000`: five-minute cap for the complete terminal exchange, including waiting for the final response. A silent financial exchange returns `RESULT_UNKNOWN` if the request may have reached the terminal.
- `connectivityLossGracePeriodMillis = 5_000`: continuous loss of the Android network interface owning the pay socket closes a sent exchange and returns `UNKNOWN` / `RESULT_UNKNOWN`. A second live Wi-Fi/Ethernet interface does not hide that loss.
- `abortTimeoutMillis = 10_000`: maximum time to await an abort response, capped by the operation timeout. An abort timeout is a communication error, and the original financial result remains authoritative.

A TCP reset/close ends the exchange as soon as the socket reports it. Network callbacks detect local interface loss; they cannot reliably detect a cable/router failure farther along a still-active network. TCP keepalive is enabled as an additional operating-system safeguard, but its probe timings are platform dependent. The operation timeout remains the upper bound for a silent unreachable terminal. Set a shorter connectivity grace period when your cashier must surface observed local network loss sooner.

Configure these values through `OpiSdkOptions` when the integration needs different limits:

```kotlin
val options = OpiSdkOptions(
    terminalHost = "192.168.0.69",
    connectTimeoutMillis = 10_000,
    operationTimeoutMillis = 300_000,
    connectivityLossGracePeriodMillis = 5_000,
    abortTimeoutMillis = 10_000
)
```

## Transaction state and reconciliation

**SDK owns OPI communication; ECR owns transaction state and reconciliation.**

The SDK keeps no persistent transaction ledger or cross-restart transaction lock. An overlapping operation is rejected with `PENDING_TRANSACTION` while an exchange is active. Save the sale intent, terminal identity, result and receipts in your ECR.

`payment()` → `PRINTLASTTICKET` → ECR decides → `reprint()` if required → ECR explicitly starts a new payment.

`PRINTLASTTICKET` is returned unchanged. The SDK does not reprint, reconcile an older sale, check readiness, or retry the new payment automatically. A successful reprint does not start another payment.

If a financial request may have been sent but its final response is lost or unusable, the SDK returns `UNKNOWN` / `RESULT_UNKNOWN`. This is neither approval nor proof of failure. Keep the sale unresolved and **do not blindly retry**: reconcile using the terminal/acquirer records and any explicitly requested evidence. A connection failure before transmission returns a communication/connection error. A definitive terminal response preserves its outcome and error code.

`repeatLastMessage()` explicitly requests the terminal's last registered message. `reprint()` explicitly requests its last receipt and available transaction details using the configured receipt handling. These are distinct commands; neither calls the other. The ECR must correlate their evidence with its own sale, especially if another ECR has used the terminal. Neither command updates an SDK ledger.

### Network interruption and receipt recovery

A terminal can approve a transaction while the cashier is offline. If the original exchange remains usable long enough to deliver a valid final approval, `SUCCESS` is the financial outcome. Missing receipts must not turn that approval into a decline. If the SDK has already ended the exchange after observed network loss or timeout, the original call returns `UNKNOWN`; reconnecting does not automatically resolve it or retry it.

Receipt presence is separate from the financial outcome. A successful `PrintLocal` reprint may legitimately contain no app receipt text. For `Available` handling, retain every nonblank returned copy and keep receipt recovery unresolved when the necessary receipt evidence is missing. The SDK preserves the terminal's result; the ECR decides whether the evidence is sufficient to reconcile its sale. Do not clear a pending sale based only on an empty reprint or successful `GetStatus`.

After interruption, retain the original sale identity, terminal identity, amount/currency, final response if present, and every receipt already captured. Use `repeatLastMessage()` explicitly for financial response evidence and `reprint()` for receipts, correlate the evidence with that sale, and keep unknown financial outcomes unresolved until matched. The demo assumes a terminal used exclusively by that cashier; shared terminals require stronger transaction matching. Repeated empty reprints need terminal-side investigation and sanitized pay/device-channel logs, not an automatic payment retry.

An `AbortRequest` acknowledgement does not prove that a financial transaction was cancelled. Keep awaiting the original financial result. If that response is lost, its outcome remains unknown; do not infer cancellation from an abort acknowledgement or closed socket.

## File logging

File logging is disabled by default. Enable it during integration or support diagnostics:

```kotlin
val opiSdk = OpiAndroidSdk.create(
    context,
    OpiSdkOptions(
        loggingEnabled = true,
        maxLogStorageBytes = 20L * 1024L * 1024L
    )
)
```

Logs are stored in the host application's private files directory:

```text
<app files>/opi-android-sdk/logs/YYYY-MM-DD.log
```

Normal logs contain correlation identifiers, operation types, connection events and classified outcomes. Extended adds sanitized OPI XML, including terminal messages and returned receipts. PAN, expiry, track data, and bearer credentials are removed. Treat the remaining content as protected operational data.

Applications can provide an internal support viewer or export flow through the SDK:

```kotlin
suspend fun prepareSupportLogs() {
    val directory = opiSdk.logDirectoryPath
    val files = opiSdk.listLogFiles()
    val content = files.firstOrNull()?.let { opiSdk.readLogFile(it.name) }

    // Use only for an explicit support/user action.
    opiSdk.clearLogFiles()
}
```

The oldest daily files are removed when the configured storage limit is reached.

### Logging levels

Logging is off by default. Set `loggingEnabled = true` to enable it and choose `logLevel = OpiLogLevel.Normal` (default) for operation summaries, or `OpiLogLevel.Extended` for sanitized OPI XML and error details. With logging off, neither level writes logs. Extended diagnostics cover SDK communication and SDK errors only. Application exceptions and crashes are outside SDK logging.

When the ECR requests `reversal()`, the SDK automatically confirms the terminal’s `GetConfirmation` prompt for that reversal. Receipt confirmations retain their existing automatic handling; this does not enable general card-consent confirmation.

`info()` sends OPI `GetInfo`. A terminal can emit diagnostic printer reports and still return `Failure`; the SDK preserves that terminal result. Use `status()` (`GetStatus`) for current terminal identity and readiness. An idle status is not proof that a pending receipt was cleared.
