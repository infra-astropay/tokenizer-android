# Tokenizer Android SDK

> Documentation updated against `main` @ `290cebf` (SDK version `1.1.0`).

## 1. Table of contents
- [1. Table of contents](#1-table-of-contents)
- [2. Overview](#2-overview)
- [3. Getting started](#3-getting-started)
- [4. Collect](#4-collect)
- [5. Revealer](#5-revealer)
- [6. Resources](#6-resources)

## 2. Overview
The AstroPay Tokenizer SDK is a tool designed to facilitate easy integration of AstroPay's secure payment solutions into your Android applications. This SDK provides a way to tokenize and reveal sensitive payment information, ensuring that your transactions are both secure and efficient.

## 3. Getting Started
* **Demo project:** AP Tokenize Demo (`:app` module, package `com.aptokenizer.demoapp`)
* **SDK module:** `:tokenizer` — package `com.aptokenizer.tokenizer`
* **Folders:**
```gradle
/data
    ApiClient.kt
    RetrofitClient.kt
    /interceptors
    /models
        /collect
        /reveal
    /repositories

/domain
    /actions
    /models
        /collect
        /reveal
    /repositories

/views
    /compose
    /core
    /models
    /system
    /utils
```

##### SDK Initializer
- `TokenizerConfig.kt` (object / singleton)

##### Entry points:

- `TokenCollect.kt`
- `TokenRevealer.kt`

### 3.1 Dependencies:

| **Dependency** | **Version** |
| --- | --- |
| *Min SDK* | 26 |
| *Compile SDK* | 37 |
| *Java source/target compatibility* | 17 |
| *Android Gradle Plugin* | 9.3.0 |
| *org.jetbrains.kotlin (KGP / stdlib)* | 2.3.10 |
| *androidx.core:core-ktx* | 1.13.1 |
| *androidx.appcompat:appcompat* | 1.7.1 |
| *com.google.android.material:material* | 1.13.0 |
| *org.jetbrains.kotlinx:kotlinx-coroutines-core* | 1.10.2 |
| *org.jetbrains.kotlinx:kotlinx-coroutines-android* | 1.10.2 |
| *com.squareup.okhttp3:okhttp* | 4.12.0 |
| *com.squareup.okhttp3:logging-interceptor* | 4.12.0 |
| *com.squareup.retrofit2:converter-gson* | 2.9.0 |

##### Compose dependencies:

| **Dependency** | **Version** |
| --- | --- |
| *androidx.compose:compose-bom* | 2026.03.00 |
| *androidx.compose.material:material* | managed by the BOM |
| *androidx.activity:activity-compose* | 1.11.0 |

💡 **You do not need to declare any of these yourself.** As of `1.1.0` the published POM is generated from the release component, so every dependency above (including the Compose BOM, imported as a platform) arrives transitively when you add the `tokenizer` dependency. This was not the case in `1.0.0`, whose POM declared no dependencies at all and required integrators to add okhttp, retrofit and appcompat by hand.

⚠️ The SDK ships Compose Material (`androidx.compose.material`, i.e. Material **2**), not Material 3. `TRTextField` is built on the M2 `TextField`, so `TextFieldColors`, `TextFieldDefaults` and `LocalTextStyle` in its signature are the M2 types.

### 3.2 How to integrate:

The SDK is served from a static Maven repository hosted on GitHub Pages. Add it to your `settings.gradle.kts` (recommended, centralized repository management):

```kotlin
dependencyResolutionManagement {
    repositories {
        google()
        mavenCentral()
        maven {
            url = uri("https://infra-astropay.github.io/tokenizer-android/")
        }
    }
}
```

This makes the tokenizer dependency available across all modules. If your project uses per-project repositories instead, add the same `maven { ... }` block to the `repositories` of the consuming module's `build.gradle.kts`.

Then declare the dependency in your module's `build.gradle.kts` and rebuild:

```kotlin
dependencies {
    implementation("com.aptokenizer:tokenizer:1.1.0")
}
```

**Available versions**

| **Version** | **Status** |
| --- | --- |
| `1.1.0` | Current release. `<latest>` and `<release>` in `maven-metadata.xml`. |
| `1.0.0` | Previous release. POM without transitive dependencies. |

⚠️ `maven-metadata.xml` also lists a `1.1.1` entry, but no `1.1.1` directory exists in the repository — requesting it, or using a dynamic selector such as `1.1.+` / `latest.release` that picks the highest listed version, fails to resolve. Pin the version explicitly.

---
## 4. Collect

### 4.1 Table of contents

- [4.2 Initialization](#42-initialization)
- [4.3 SDK functions](#43-sdk-functions)
- [4.4 Components](#44-components)
    - [4.4.1 For Jetpack Compose](#441-for-jetpack-compose)
    - [4.4.2. For Android Views](#442-for-android-views)
- [4.5 Instructions](#45-instructions)

### 4.2 Initialization

Initialize the SDK once, before using any entry point, by calling `TokenizerConfig.init()`:

```kotlin
import com.aptokenizer.tokenizer.TokenizerConfig
import com.aptokenizer.tokenizer.domain.models.Environment

class YourApplication : Application() {
    override fun onCreate() {
        super.onCreate()
        TokenizerConfig.init(
            environment = Environment.Sandbox,
            seeLogs = true
        )
    }
}
```

`Environment` is an enum in `com.aptokenizer.tokenizer.domain.models` with two entries: `Production` and `Sandbox`. The base URL is selected from it — there is no way to override it from the host app.

`TokenizerConfig.isInitialized` reports whether `init()` has already run. Calling `collect()` or `reveal()` before initialization throws `IllegalStateException("SDK not initialized. Call init() first.")`.

💡 Use the Sandbox environment for testing only. We recommend not allowing logs in the production release of the app.

### 4.3 SDK Functions

**TokenizerConfig:**

- **`init(environment: Environment, seeLogs: Boolean = false, timeout: Long = 60)`** → Initializes the SDK dependencies: the target environment, whether HTTP bodies should be logged to the console, and the OkHttp connect/read/write timeout in **seconds**.
- **`isInitialized: Boolean`** → whether `init()` has already been called.

**TokenCollect:**

- **`setAccessToken(accessToken: String)`** → sets a valid authentication token to consume SDK functions.
- **`clearData()`** → clears both the registered field values and the tokens already collected by this instance.
- **`collect(trResult: (TRResult) -> Unit)`** → takes the data from the list of fields registered with the respective instance and sends them to be tokenized. Throws `IllegalStateException` if the SDK was not initialized or if no access token was set.

`TokenCollect.TRResult` is a sealed class with five cases:

```kotlin
TRResult.Success(listTokenizedData: Map<String, String>)
TRResult.Error(error: String, message: String)
TRResult.NetworkError
TRResult.InvalidToken(message: String)
TRResult.ValueNoValid(errorMessage: String)
```

### 4.4 Components

##### **4.4.1 For Jetpack Compose**

* ##### **TRTextField(..)** → Composable that allows the entry of data to be collected

**Mandatory parameters:**

1. **`tokenCollect: TokenCollect`** the instance that will be listening to the information entered by the user
2. **`fieldName: String`** is an identifier for the field, with which you can then access the respective token in the returned list.

**All parameters** (listed in declaration order):

| **Parameter** | **Type** | **Default** | **Description** |
| --- | --- | --- | --- |
| `tokenCollect` | `TokenCollect` | - | It is the instance that will be listening to the information entered by the user |
| `modifier` | `Modifier` | `Modifier` | Modifiers allow you to decorate or augment a composable. |
| `fieldName` | `String` | - | It is an identifier for the field, with which you can then access the respective token in the returned list |
| `minLength` | `Int?` | `null` | Minimum number of characters the field accepts. Input shorter than this is not recorded. |
| `maxLength` | `Int?` | `null` | Maximum number of characters the field accepts. **Ignored** when `textInputType` is `CARD_NUMBER`, `EXPIRATION_DATE` or `CVV` — see the note below. |
| `onFieldStateChange` | `((FieldState) -> Unit)?` | `null` | Lambda with which you can listen to properties related to the value entered in the field |
| `enabled` | `Boolean` | `true` | Controls the enabled state of the TextField. When false, the text field will be neither editable nor focusable, and its input will not be selectable. |
| `readOnly` | `Boolean` | `false` | Controls the editable state of the TextField. When true, the text field can't be modified, but a user can focus it and copy text from it. |
| `textStyle` | `TextStyle` | `LocalTextStyle.current` | The style to be applied to the input text. |
| `label` | `@Composable (() -> Unit)?` | `null` | Optional label to be displayed inside the text field container. |
| `placeholder` | `@Composable (() -> Unit)?` | `null` | Optional placeholder displayed when the text field is focused and the input text is empty. |
| `leadingIcon` | `@Composable (() -> Unit)?` | `null` | Optional leading icon displayed at the beginning of the text field container. |
| `trailingIcon` | `@Composable (() -> Unit)?` | `null` | Optional trailing icon displayed at the end of the text field container. Overridden by the brand image when `textInputType == CARD_NUMBER` and `showBrand` is true. |
| `showBrand` | `Boolean` | `false` | When true, if `textInputType` is `CARD_NUMBER`, shows the detected card brand image as the trailing icon. |
| `isRequired` | `Boolean` | `false` | Marks the field as required, which participates in the `isValid` computation of `FieldState`. |
| `enableTokenization` | `Boolean` | `true` | Requests tokenization for this field. **Ignored** for `CARD_NUMBER` and `CVV`, which are always tokenized. |
| `volatile` | `Boolean` | `false` | Requests volatile (non-durable) storage on the backend side. Can only **raise** volatility: `CVV` is always volatile regardless of this value. |
| `isError` | `Boolean` | `false` | Indicates the current value is in error. When true, label, bottom indicator and trailing icon are displayed in the error color. |
| `textInputType` | `TextInputType` | `TextInputType.NONE` | Expected type of input. Options: `NONE`, `CARD_NUMBER`, `EXPIRATION_DATE`, `CVV`, `PASSWORD`. Drives masking, input filtering, length limits, tokenization and volatility. |
| `keyboardOptions` | `KeyboardOptions` | `KeyboardOptions.Default` | Software keyboard options such as KeyboardType and ImeAction. |
| `keyboardActions` | `KeyboardActions` | `KeyboardActions.Default` | Callbacks invoked when the input service emits an IME action. |
| `singleLine` | `Boolean` | `true` | When true, the field becomes a single horizontally scrolling line and `maxLines` is ignored. |
| `maxLines` | `Int` | `Int.MAX_VALUE` | Maximum number of visible lines. Ignored when `singleLine` is true. |
| `interactionSource` | `MutableInteractionSource` | `remember { MutableInteractionSource() }` | The MutableInteractionSource representing the stream of Interactions for this TextField. |
| `shape` | `Shape` | `TextFieldDefaults.TextFieldShape` | The shape of the text field's container. |
| `colors` | `TextFieldColors` | `TextFieldDefaults.textFieldColors()` | Colors used to resolve text, content and background in the different states. |
| `onScrollToPosition` | `((yPosition: Int) -> Unit)?` | `null` | Invoked with the field's y position in the root when it gains focus, so the host can scroll to it. |
| `regexRulesValidation` | `List<RegexRuleValidation>?` | `null` | Rules applied to the value entered by the user. Each item carries a `pattern` and an optional `errorMessage`. Results are delivered both in `onFieldStateChange` and as a response in `collect()`. |

⚠️ **Length limits for typed fields are derived, not configured.** When `textInputType` is `CARD_NUMBER` or `CVV`, the effective maximum comes from the detected brand (`BrandAttributes.rangeNumber` / `rangeCVV`); for `EXPIRATION_DATE` it is fixed at 6 digits. Your `maxLength` is only honoured for `NONE` and `PASSWORD`.

⚠️ **Security floors.** `enableTokenization` and `volatile` are *requests*, not switches. Field classification always wins: `CARD_NUMBER` and `CVV` are tokenized whatever you pass, and `CVV` is always stored as volatile, because a card verification code must not be retained after the transaction it was collected for.

**FieldState**

`FieldState` is a sealed class. `CARD_NUMBER` fields emit `FieldState.CardNumber`, everything else emits `FieldState.TextField`.

| **Property** | **Type** | **Present in** |
| --- | --- | --- |
| `contentLength` | `Int` | both |
| `hasFocus` | `Boolean` | both |
| `isEmpty` | `Boolean` | both |
| `isValid` | `Boolean` | both |
| `regexRuleValidationResult` | `List<RegexRuleValidationResult>` | both |
| `textInputType` | `TextInputType` | both |
| `bin` | `String` | `CardNumber` only |
| `last` | `String` | `CardNumber` only — last 4 digits |
| `cardBrand` | `String` | `CardNumber` only |

For `CARD_NUMBER`, `isValid` additionally requires the brand to be recognized and, when the brand uses it, the Luhn checksum to pass.

##### **4.4.2 For Android Views**

* ##### **TREditText** → `TextInputEditText` wrapper that allows the entry of data to be collected. Fully qualified name: `com.aptokenizer.tokenizer.views.system.TREditText`.

**Mandatory setup:**

1. **`setTokenCollect(tokenCollect: TokenCollect)`** — the instance that will be listening to the information entered by the user. This is **not** an XML attribute; it must be set from code. Typing into the field before it is set throws `IllegalStateException("TokenCollect instance not set. Call setTokenCollect() first.")`.
2. **`app:fieldName`** (or `setFieldName(...)`) — an identifier for the field, with which you can then access the respective token in the returned list.

**All XML attributes:**

| **Attribute** | **Format** | **Default** | **Description** |
| --- | --- | --- | --- |
| `app:hint` | string | `null` | Text displayed while the field is empty. |
| `app:hintTextColor` | color | `Color.GRAY` | Color of the hint text. |
| `app:fieldName` | string | `""` | Identifier for the field, used to retrieve the respective token from the returned list. |
| `android:gravity` | int | `start\|center_vertical` | Horizontal and vertical alignment of the text. |
| `app:singleLine` | boolean | `false` | Whether the field is displayed on a single line. |
| `app:textSize` | dimension | unset | Font size. |
| `app:enabled` | boolean | `true` | Enabled state of the field. |
| `app:textColor` | color | `Color.BLACK` | Font color. |
| `android:textStyle` | int | `Typeface.NORMAL` | Normal / bold / italic. |
| `app:fontFamily` | string / reference | unset | Font family. |
| `app:letterSpacing` | float | unset | Letter spacing in em units. |
| `android:inputType` | int | `TYPE_CLASS_TEXT` | Content type as defined for `EditorInfo.inputType`. |
| `app:textAppearance` | reference | `0` | Text appearance style resource. |
| `app:maxLength` | integer | `Int.MAX_VALUE` | Maximum number of characters. Derived from the field type for `card_number`, `expiration_date` and `cvv`. |
| `app:minLength` | integer | `0` | Minimum number of characters for the value to be considered valid. Derived from the brand for `cvv`. |
| `app:isRequired` | boolean | `false` | Marks the field as required for validation purposes. |
| `app:showBrand` | boolean | `false` | Shows the detected card brand image. Only meaningful with `textInputType="card_number"`. |
| `app:enableTokenization` | boolean | `true` | Requests tokenization. Ignored for `card_number` and `cvv`, which are always tokenized. |
| `app:volatileField` | boolean | `false` | Requests volatile storage on the backend side. Can only raise volatility — `cvv` is always volatile. |
| `app:textInputType` | enum | `none` | One of `none`, `card_number`, `expiration_date`, `cvv`, `password`. |

⚠️ The `app:textInputType` XML enum is resolved **by ordinal** against `TextInputType`, so the XML values map positionally: `none`=0, `card_number`=1, `expiration_date`=2, `cvv`=3, `password`=4.

**Public functions:**

- `setTokenCollect(tokenCollect: TokenCollect)` — the instance that will be listening to the information entered by the user. Also primes the brand image and the length limits.
- `setOnFieldStateChangeListener(listener: OnFieldStateChangeListener?)` — callback that allows listening to properties related to the value entered in the field. The listener receives the same `FieldState` described in 4.4.1:

```kotlin
contentLength
hasFocus
isEmpty
isValid
regexRuleValidationResult
textInputType
```

- `setFieldName(name: String)`
- `setTextInputType(textInputType: TextInputType)` — applies the corresponding mask and input filtering. Use `TextInputType.CARD_NUMBER` to group the digits according to the detected brand's mask; `PASSWORD`, `EXPIRATION_DATE` and `CVV` are also available.
- `setRegexPattern(regexRuleValidation: List<RegexRuleValidation>)` — the Android Views equivalent of `regexRulesValidation`.
- `setError(message: String)` — the message shown on the field once its state becomes invalid.
- `setMaxLength(textMaxLength: Int)` / `setMinLength(textMinLength: Int)`
- `setSelection(index: Int)` / `setSelection(start: Int, stop: Int)`
- `clearText()`
- Styling passthroughs: `setGravity`, `setHint`, `setHintTextColor`, `setTextColor`, `setTextSize`, `setTextAppearance`, `setTypeface`, `getTypeface`, `setSingleLine`, `setInputType`, `setLetterSpacing`.

### 4.5 Instructions

Create the composition views needed for tokenization, one for each data to be tokenized, for example:

```kotlin
TRTextField(
    tokenCollect = tokenCollect,
    modifier = Modifier.fillMaxWidth(),
    fieldName = cardFieldName,
    enabled = tokenSaved,
    isRequired = true,
    showBrand = true,
    textInputType = TextInputType.CARD_NUMBER,
    keyboardOptions = KeyboardOptions.Default.copy(
        keyboardType = KeyboardType.Number
    ),
    onFieldStateChange = { fieldState ->
        isValidCardNumber = fieldState.isValid
    },
    placeholder = {
        Text(text = "Enter the 16 digits of the card")
    }
)
```

A field with custom validation rules, for example a PIN:

```kotlin
TRTextField(
    tokenCollect = tokenCollect,
    modifier = Modifier.fillMaxWidth(),
    fieldName = pinFieldName,
    isError = errorPin.isNotEmpty(),
    keyboardOptions = KeyboardOptions.Default.copy(
        keyboardType = KeyboardType.NumberPassword
    ),
    onFieldStateChange = { fieldState ->
        isValidPin = fieldState.isValid
        errorPin.clear()
        fieldState.regexRuleValidationResult.forEach { validation ->
            if (validation.isValid.not()) {
                validation.errorMessage?.let { errorPin.add(it) }
            }
        }
    },
    regexRulesValidation = listOf(
        RegexRuleValidation(
            pattern = "^(?!.*(\\d)\\1{3}).*$",
            errorMessage = "Repeated digits are not allowed."
        )
    )
)
```

Or create the Android View in your XML file, example:

```xml
<com.aptokenizer.tokenizer.views.system.TREditText
    android:id="@+id/tr_text_number_card"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:layout_marginHorizontal="16dp"
    android:layout_marginTop="16dp"
    android:background="@drawable/tr_edit_text_bg"
    android:inputType="number"
    android:paddingHorizontal="4dp"
    android:paddingVertical="6dp"
    app:enabled="false"
    app:fieldName="number"
    app:hint="@string/card_digits_hint"
    app:hintTextColor="@color/text_muted"
    app:isRequired="true"
    app:showBrand="true"
    app:textColor="@color/white"
    app:textInputType="card_number"
    app:textSize="16sp"
    app:layout_constraintStart_toStartOf="parent"
    app:layout_constraintTop_toTopOf="parent" />
```

```kotlin
val cardEditText = findViewById<TREditText>(R.id.tr_text_number_card)
```

**Create the TokenCollect instance:**

You need to create the TokenCollect instance for the respective flow of your application in which you will tokenize the respective data.

```kotlin
val tokenCollect = TokenCollect()
```

**Set a valid authorization token:**

Set a valid authorization token in the SDK (`tokenCollect.setAccessToken`) to be able to use the tokenization functions. Generally it will be your backend team who provides it to you: see backend documentation [here](https://www.notion.so/Guidelines-for-Merchant-and-Card-Issuers-eba6790d685b44be90b16fae1447cc38?pvs=21).

```kotlin
tokenCollect.setAccessToken("authorizationToken")
```

You can use the following [site](https://dinochiesa.github.io/jwt/) to generate **test jwt tokens** by providing the data provided to your backend team by our team.

**Collect the respective data:**

Now you just need to call `tokenCollect.collect()`, which tokenizes all the registered fields. To know the result, pass a lambda function, like this:

```kotlin
tokenCollect.collect { result ->
    cardToken = String()
    pinToken = String()
    when (result) {
        is TokenCollect.TRResult.Success -> {
            cardToken = result.listTokenizedData[cardFieldName] ?: String()
            pinToken = result.listTokenizedData[pinFieldName] ?: String()
        }

        is TokenCollect.TRResult.Error -> {
            var errorMessage = result.error
            if (result.message.isNotEmpty()) errorMessage = errorMessage.plus(" - ${result.message}")
            showMessage(errorMessage)
        }
        is TokenCollect.TRResult.NetworkError -> showMessage("Network Error")
        is TokenCollect.TRResult.InvalidToken -> showMessage("Invalid Token")
        is TokenCollect.TRResult.ValueNoValid -> showMessage(result.errorMessage)
    }
    loading = false
}
```

⚠️ The callback is invoked from a background coroutine (`Dispatchers.IO`), not from the main thread. Post to the main dispatcher before touching the UI directly.

💡 Fields registered with `enableTokenization = false` (and not classified as cardholder data) are returned in `listTokenizedData` **as plain text** under their `fieldName`, not as tokens.

##### **For Android Views:**

It is important that you register the TokenCollect instance that will listen for changes in the respective text field for it to work properly:

```kotlin
trEditText.setTokenCollect(tokenCollect)
```

Don't forget to define a unique fieldName for each field in your form; with this you can retrieve the respective token from the list provided.

---
## 5. Revealer

### Table of contents

- [5.1 Initialization](#51-initialization)
- [5.2 SDK functions](#52-sdk-functions)
- [5.3 Components](#53-components)
    - [5.3.1 For Jetpack Compose](#531-for-jetpack-compose)
    - [5.3.2 For Android Views](#532-for-android-views)
- [5.4 Instructions](#54-instructions)
- [5.5 Other functions](#55-other-functions)

### 5.1 Initialization

The Revealer shares the same initialization as Collect — a single `TokenizerConfig.init()` call covers both:

```kotlin
import com.aptokenizer.tokenizer.TokenizerConfig
import com.aptokenizer.tokenizer.domain.models.Environment

class YourApplication : Application() {
    override fun onCreate() {
        super.onCreate()
        TokenizerConfig.init(
            environment = Environment.Sandbox,
            seeLogs = true
        )
    }
}
```

💡 Use the Sandbox environment for testing only. We recommend not allowing logs in the production release of the app.

### 5.2 SDK Functions

**TokenizerConfig:**

- **`init(environment: Environment, seeLogs: Boolean = false, timeout: Long = 60)`** → Initializes the SDK dependencies: the target environment, whether HTTP bodies should be logged, and the timeout in seconds.

**TokenRevealer** (instance methods — create one with `TokenRevealer()`):

- **`subscribe(contentPath: String, token: String, textView: TRTextView? = null)`** → subscribes a field that requires to be revealed. Pass `textView` only when using Android Views; for Compose, `TRText` looks the value up by `contentPath`.
- **`unsubscribe(contentPath: String)`** → removes the indicated field from the list of tokens to reveal, and drops any value already revealed for it.
- **`setAccessToken(accessToken: String)`** → sets a valid authentication token to consume SDK functions.
- **`reveal(trResult: (TRResult) -> Unit)`** → takes the list of subscribed fields, reveals them, and renders them to the corresponding view. Throws `IllegalStateException` if the SDK was not initialized or if no access token was set.
- **`copy(context: Context, contentPath: String)`** → copies the corresponding revealed data to the device's clipboard.
- **`clearData()`** → clears both the subscriptions and the revealed data.

`TokenRevealer.TRResult` is a sealed class with four cases:

```kotlin
TRResult.Success
TRResult.Error(error: String, message: String)
TRResult.NetworkError
TRResult.InvalidToken(message: String)
```

### 5.3 Components

##### **5.3.1 For Jetpack Compose**

* ##### **TRText(..)** → Composable that renders the revealed data.

**Mandatory parameters:**

1. **`tokenRevealer: TokenRevealer`** the instance in which you are subscribing to the tokens to be revealed
2. **`hintText: String`** the text to be displayed while the data has not been revealed
3. **`contentPath: String`** the key that must match the token subscribed with the `subscribe(...)` function

**Optional attributes:**

1. **`showDataRevealed`** that allows you to show or hide the revealed data, plus the styling parameters below.

**All parameters** (listed in declaration order):

| **Parameter** | **Type** | **Default** | **Description** |
| --- | --- | --- | --- |
| `tokenRevealer` | `TokenRevealer` | - | It is the instance in which you are subscribing to the tokens to be revealed |
| `modifier` | `Modifier` | `Modifier` | Modifiers allow you to decorate or augment a composable. |
| `hintText` | `String` | - | Text displayed while the data is not revealed. |
| `contentPath` | `String` | - | Identification key of the respective token that will be displayed in this view. |
| `showDataRevealed` | `Boolean` | `false` | Hides or shows previously revealed data. While false — or before `reveal()` succeeds — `hintText` is rendered. |
| `color` | `Color` | `Color.Unspecified` | Font color. |
| `fontSize` | `TextUnit` | `TextUnit.Unspecified` | Font size. |
| `fontWeight` | `FontWeight?` | `null` | Font weight. |
| `fontFamily` | `FontFamily?` | `FontFamily.Default` | Font family. |
| `textAlign` | `TextAlign?` | `null` | Text alignment. |
| `lineHeight` | `TextUnit` | `TextUnit.Unspecified` | Line height. |
| `overflow` | `TextOverflow` | `TextOverflow.Clip` | Text overflow behavior. |
| `softWrap` | `Boolean` | `true` | Whether the text should break at soft line breaks. |
| `maxLines` | `Int` | `Int.MAX_VALUE` | Optional maximum number of lines for the text to span, wrapping if necessary. |
| `onTextLayout` | `(TextLayoutResult) -> Unit` | `{}` | Callback executed when a new text layout is calculated. |
| `style` | `TextStyle` | `TextStyle.Default` | Style configuration for the text such as color, font, line height etc. |
| `regexReplaceField` | `RegexReplaceField?` | `null` | Two components: the **pattern**, the regular expression matched against the revealed text, and the **replacement**, what each match is replaced with. Both must be non-null for the replacement to apply. |

##### **5.3.2 For Android Views**

* ##### **TRTextView** → TextView wrapper where the revealed data will be displayed. Fully qualified name: `com.aptokenizer.tokenizer.views.system.TRTextView`.

**Mandatory setup:**

1. **`app:hint`** the text to be displayed while the data has not been revealed.
2. The view instance must be handed to `tokenRevealer.subscribe(contentPath = ..., token = ..., textView = ...)`. That call — not an XML attribute — is what binds the view to a `contentPath`.

**All XML attributes:**

| **Attribute** | **Format** | **Default** | **Description** |
| --- | --- | --- | --- |
| `app:hint` | string | `null` | Text displayed while the data is not revealed. |
| `app:hintTextColor` | color | `Color.GRAY` | Color of the hint text. |
| `app:contentPath` | string | — | **Declared but not read by the view.** See the note below. |
| `android:gravity` | int | `start\|center_vertical` | Horizontal and vertical alignment of the text. |
| `app:singleLine` | boolean | `false` | Whether the field is displayed on a single line. |
| `app:textSize` | dimension | unset | Font size. |
| `app:enabled` | boolean | `true` | Enabled state of the view. |
| `app:textColor` | color | `Color.BLACK` | Font color. |
| `android:textStyle` | int | `Typeface.NORMAL` | Normal / bold / italic. |
| `app:fontFamily` | string / reference | unset | Font family. |
| `app:letterSpacing` | float | unset | Letter spacing in em units. |
| `android:inputType` | int | `TYPE_NULL` | Content type as defined for `EditorInfo.inputType`. |
| `app:textAppearance` | reference | `0` | Text appearance style resource. |
| `app:regexPattern` | string | `null` | Regular expression matched against the revealed text. |
| `app:regexReplacement` | string | `null` | Replacement applied to each match. Both `regexPattern` and `regexReplacement` must be present for the replacement to apply. |

⚠️ `app:contentPath` is declared in the styleable but `TRTextView` never reads it, so setting it has no effect — it appears in the demo layouts as documentation only. The token-to-view mapping is established exclusively by the `textView` argument of `subscribe(...)`.

**Public functions:**

- `showDataRevealed(show: Boolean)` — swaps between the revealed value and the hint. This is the Android Views equivalent of the `showDataRevealed` parameter of `TRText`.
- `clearText()`
- Styling passthroughs: `setGravity`, `setHint`, `setHintTextColor`, `setTextColor`, `setTextSize`, `setTextAppearance`, `setTypeface`, `getTypeface`, `setSingleLine`, `setInputType`, `setLetterSpacing`.

💡 Neither `TRTextView` nor `TREditText` exposes a getter for the underlying value — the revealed data is never readable from your application code, which keeps the host app out of PCI scope.

### 5.4 Instructions

Create the Compose view(s) needed for revelation, one for each token to be revealed, example:

```kotlin
TRText(
    tokenRevealer = tokenRevealer,
    modifier = Modifier
        .weight(1f)
        .height(35.dp),
    contentPath = contentPath,
    hintText = "• • • •  • • • •  • • • •  • • • •",
    showDataRevealed = show,
    fontSize = 18.sp,
    regexReplaceField = RegexReplaceField(
        pattern = "(\\d{4})(?=\\d)",
        replacement = "$1 "
    )
)
```

Or create the Android View in your XML file, example:

```xml
<com.aptokenizer.tokenizer.views.system.TRTextView
    android:id="@+id/card_text_view"
    android:layout_width="0dp"
    android:layout_height="wrap_content"
    android:layout_marginHorizontal="16dp"
    android:layout_marginTop="16dp"
    app:hint="• • • •  • • • •  • • • •  • • • •"
    app:hintTextColor="@color/white"
    app:regexPattern="(\\d{4})(?=\\d)"
    app:regexReplacement="$1 "
    app:textColor="@color/white"
    app:textSize="18sp"
    app:layout_constraintEnd_toEndOf="parent"
    app:layout_constraintStart_toStartOf="parent"
    app:layout_constraintTop_toTopOf="parent" />
```

```kotlin
val cardTextView = findViewById<TRTextView>(R.id.card_text_view)
```

**Create the TokenRevealer instance:**

You need to create the TokenRevealer instance for the respective flow of your application in which you will subscribe to the respective tokens.

```kotlin
val tokenRevealer = TokenRevealer()
```

**Set a valid authorization token:**

Set a valid JWT authorization token in the **SDK** (`tokenRevealer.setAccessToken`) to be able to use the reveal functions. Generally it will be your backend team who provides it to you: see backend documentation [here](https://www.notion.so/Guidelines-for-Merchant-and-Card-Issuers-eba6790d685b44be90b16fae1447cc38?pvs=21).

```kotlin
tokenRevealer.setAccessToken("authorizationToken")
```

**How to generate a JWT authentication token**

Every request must be authenticated using a JWT token that is securely signed with your private key.

Since a private key is essential for signing JWT tokens, and distributing these keys directly to mobile devices poses significant security risks, we recommend implementing a back-end service dedicated to securely generating JWT tokens for your mobile applications.

💡 You can use the following [site](https://dinochiesa.github.io/jwt/) to generate **test jwt tokens** by providing the data provided to your backend team by our team.

**Subscribe the tokens to be revealed:**

Subscribe the tokens to be revealed in the required section of your app.

```kotlin
tokenRevealer.subscribe(
    contentPath = "contentPathExample",
    token = "tok_test_dfi3uAtS02KyeoQ2ja2C7Fd8MXe84MBd123",
    textView = cardTextView // Only if you use Android Views
)
```

💡 Keep in mind that the **contentPath** is a string defined as a key to then identify the view where the subscribed token information will be revealed. The **token** to reveal is usually provided by your backend.

**Reveal the respective data:**

Now you just need to call `tokenRevealer.reveal()`, which reveals all the subscribed data. To know the result, pass a lambda function, like this:

```kotlin
tokenRevealer.reveal { result ->
    when (result) {
        is TokenRevealer.TRResult.Success -> // do something

        is TokenRevealer.TRResult.Error -> {
            result.error
            // do something
        }

        is TokenRevealer.TRResult.NetworkError -> // do something

        is TokenRevealer.TRResult.InvalidToken -> // do something
    }
}
```

⚠️ As with `collect()`, the callback runs on a background coroutine (`Dispatchers.IO`). Post to the main dispatcher before touching the UI directly.

### 5.5 Other functions:

**Copy function:**

Use the `copy()` function to copy the revealed information to the clipboard. Keep in mind that you will never have access to the information directly from your application, to avoid being within reach of PCI.

```kotlin
tokenRevealer.copy(context, contentPath)
```

💡 This function receives as parameters the current **context** and the **contentPath** string that was previously subscribed.

**Unsubscribe function:**

Use the `unsubscribe()` function to remove a respective token from the token list.

```kotlin
tokenRevealer.unsubscribe("contentPath")
```

💡 This function receives as a parameter the contentPath string that was previously subscribed.

**Clear data function:**

Use `clearData()` to drop every subscription and every revealed value held by the instance.

```kotlin
tokenRevealer.clearData()
```

## 6. Resources:

- **AP Tokenizer SDK (`:tokenizer`)** → provides an API for interacting with the AP Tokenizer Vault. Published as `com.aptokenizer:tokenizer`.
- **AP Tokenizer DEMO (`:app`)** → sample application that acts as the host app for testing the **SDK** during development. It covers Compose, Android Views and mixed flows for both Collect and Revealer.
