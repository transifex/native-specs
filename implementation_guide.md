# 2. Implementation guide

This section describes the various parts of a Native implementation in detail, providing all the necessary details for implementing them.

Also, in order to make the process easier, it presents a *suggested plan* on what to build first and where to go after each step. This plan was created in a way to allow developers to build the SDK in iterations and have a working software in every step.

You may follow this plan or come up with any other process that suits you best.

* [2.1 Core functionality](#21-core-functionality)
* [2.2 String marking & rendering](#22-string-marking-&-rendering)
* [2.3 Phrase extraction](#23-phrase-extraction)
* [2.4 Pushing source content](#24-pushing-source-content)
* [2.5 Daemon](#25-daemon)
* [2.6 Additional features](#26-additional-features)
* [2.7 Configuration](#27-configuration)

## 2.1 Core functionality

This part is framework-agnostic. For example, it only includes code in Python, but doesn’t deal with Django-specifics.

Start with the **Core** functionality. Focus on creating an end-to-end flow that gets a source string with optional variable values and returns its final rendered form in any language, by using the translations found in **Cache**. For now you can use a very simple cache with hard-coded content, e.g. a simple dictionary/map structure.

Ignore **MissingPolicy** and **ErrorPolicy** completely for now. If a translation is missing you can fall back to the source string in a hard-coded way.

You will need to implement [Key Generation](#heading=h.vtasjxv2sbj4), so that you can properly add content to and retrieve content from the cache, as well as **Plural handling**, in order to be able to properly store and retrieve pluralized strings.

Also, you will need to find a library that supports ICU Message Format strings, or implement this functionality on your own.

### Expected result

When this step is over you should be able to manually run a function that gets an ICU source string as input and returns a final rendered string on any language as output, using a fake/hard-coded cache as a storage for translations.

#### Pseudocode

##### MemoryCache

```javascript
// A basic cache implementation that keeps translations in memory
class MemoryCache

  function init()
    // Structure:
    // {
    //   '<locale1>': {
    //     '<key1>': {'string': '<translation1>'},
    //     '<key2>': {'string': '<translation2>'},
    //   },
    //   '<locale2>': {
    //     '<key2>': {'string': '<translation2>'},
    //     '<key3>': {'string': '<translation3>'},
    //   },
    //   ...
    // }
    this.translations_by_locale = {}

  // Return the translation for the given locale and key
  // or null if none exists
  function get(String key, String locale_code) -> String
    return this.translations_by_locale
      .get(locale_code, default={})
      .get(key, default={})
      .get('string', default=null)

  // Replace the translations for a locale with the given ones
  // See the constructor for the structure of `translations`
  function update(String locale_code, Dictionary translations)
    this.translations_by_locale[locale_code] = translations
```
            
##### NativeCore

```javascript
// Orchestrates the core functionality of the SDK
class NativeCore

  function init(Array[String] locale_codes, String token, String secret=null, String cds_host=null, Cache cache=null)
    this.locale_codes = locale_codes
    this.token = token
    this.secret = secret
    this.cds_host = cds_host
    this.cache = cache or MemoryCache()

  function translate(String source_string, String locale_code, Dictionary params, String context=None, bool is_source=False) -> String
    if is_source
      translation_template = source_string
    else
      key = generate_key(source_string, context)
      translation_template = this.cache.get(key, locale_code)

    return render_string(source_string, translation_template, locale_code, params)
```

##### StringRenderer

```javascript
// Takes care of rendering a string, by replacing any placeholders 
// with actual variable values. Later it will also handle
// the case of missing or invalid translations
function render_string(String source_string, String string_to_render, String locale_code, Dictionary params) -> String
  if not string_to_render
    // hard-coded missing policy, falls back to source string
    string_to_render = source_string

  // Ignore any exceptions for the moment
  return ICUMessageFormat.format(string_to_render, params, locale_code)
```

Note that `ICUMessageFormat` is supposed to be a 3rd-party library that knows how to render ICU strings with their parameters, based on the given locale. If such a library does not exist for the programming language you are developing on, or if you don’t want to use such a library for any reason, you need to implement one of your own. 

Here is an [example integration](https://github.com/rolepoint/pyseeyou/blob/master/pyseeyou/grammar.py) of the grammar you will have to create. Also, check [how ICU works](https://format-message.github.io/icu-message-format-for-translators/index.html) to better understand exactly what should be implemented. Of all the features shown here *date formatting* is nice to have, while the rest are expected to be supported in all Native SDKs (variable placeholders, `plural`, `select`).

##### Key generation

```javascript
// Return a unique key based on the given source string and optional context.
// A string can be associated with multiple context values, so the context
// argument can be a serialized comma-separated string or a single string.
function generate_key(String source_string, String context=null) -> String
  if context
    context = context.replace(',', ':')
  if not context
    context = ''

  // Ignore plurals for now. 5 corresponds to the "other" plural rule,
  // which means that this string is treated as non-pluralized
  return md5('5:' + source_string + ':' + context)
```

The method above completely ignores plurals. Since [proper plural handling](#heading=h.wsoujvq0iil) is complex, it is suggested that you deal with that in a later stage.

#### Unit test data
##### Key generation

```javascript
generate_key('This is a nice phrase', 'some context')
// returns "1cef92f8501667684496766587608795"
generate_key('This is a nice phrase', 'context1,context2,context2')
// return "ba7e05eb67b7854841bef07de562e618"
```

##### Entry point
```javascript
native = NativeCore()

// Add hard-coded cache data until CDSHandler is implemented
native.cache.update({
  'de': {
    // 'This is a butterfly', context='some context'
    '4ffa67a7b40f7d50816524fbce71d4f0': {'string': 'Das ist ein Schmetterling'},
    // 'Welcome back, {username}', context=''
    '2eead63da34f3f1658f2da02e5f8f30d': {'string': 'Willkommen zurück, {username}'},
  }
})
native.translate('This is a butterfly', 'de', context='some context')
// returns 'Das ist ein Schmetterling', as found in the cache

native.translate('Welcome back, {username}', 'de', params={'username': 'Joe'})
// returns 'Willkommen zurück, Joe', as found in the cache
// plus replacing the `username` variable with its value

native.translate('This is a python', 'de')
// returns 'This is a python', translation does not exist in cache,
// falls back to source string
```

## 2.2 String marking & rendering

Assuming you want to create a framework-specific implementation and not just a base language implementation, a next step could be to implement the marking & rendering functionality for a specific framework, i.e. implement the t function.

Also, you will have to implement **LangState**, so that Native knows what language to render to. Depending on the framework you are developing for, you may be able and prefer to use the language component of the framework’s existing localization infrastructure, so that you don’t build everything from scratch. This is [what we did for Django](https://github.com/transifex/transifex-python/blob/devel/transifex/native/django/utils/__init__.py#L22-L23) (`settings.locale_code`, `to_locale()` and `get_language()` are part of Django’s i18n framework), to take advantage of the framework’s support of current locale based on the user session.

### Expected result

When this step is over you should be able to display strings to the user through the specific framework (e.g. React), that have been rendered through template/code marking.

### Pseudocode

#### LangState

Represented here as a static class, this is an entity that keeps track of locale-based information throughout the application. It doesn’t have to be a class and it doesn’t have to be static; It can be implemented in any way it makes it globally available in the app and ensures there is only going to be one copy of these values.

```javascript
// Keeps track of the locale-related information for the application,
// such as locales defined in the application, source and current locale.
static class LangState

  static source_locale: String = "en"
  static current_locale = "en"
  static app_locales = ["en"]

  static function init(String source_locale, Array[String] app_locales)
    this.source_locale = source_locale
    this.app_locales = app_locales

  static function is_source(String locale) -> bool
    return locale == source_locale

  static function is_current(String locale) -> bool
    return locale == current_locale
}
```

The developers that integrate your SDK will have to access this entity as early as possible in the app lifecycle and set the proper values. Alternatively, you could create a mechanism that uses some sort of environment variables and sets the proper values automatically.

In any case, for applications such as mobile apps or React web apps that retrieve all content from frontend code, whenever the user changes the current locale from the UI, a call like this needs to be made: `LangState.current_locale = <new_value>`.

#### Core initializer

```javascript
// Create instance of Native Core with no parameters
// Initialization needs to be called externally
static class TxNative
  static tx = NativeCore()

  static function initialize(Array[String] languages, String token, String secret=null, String cdsHost=null, Cache cache=null)
    if tx.initialized
      return
    tx.initialize(
      languages: languages,
      token: token,
      secret: secret,
      cdsHost: cdsHost,
      cache: cache
    )
  }

  static function translate(sourceString: String, languageCode: String, params: [String: Any], context: String = null, isSource: Bool = false) -> String
    return tx.translate(
      sourceString: sourceString,
      languageCode: languageCode,
      params: params,
      context: context,
      isSource: isSource
    )
  }
}
```

#### Global t function

This function should be created in a place where it is as accessible as possible. It will be the point of entry for handling marked localizable strings in the code and in templates, so it is best that this can be imported easily.

```javascript
function t(String string, String context=null, Dictionary params={}) -> String
  return TxNative.translate(
    sourceString: string,
    languageCode: NativeLocales.currentLocale,
    params: params,
    context: context,
    isSource: NativeLocales.isSource(NativeLocales.currentLocale)
  )
}
```

Note that this code will not do any escaping in the resulting string or the parameters. Escaping will be covered in the [corresponding section](#heading=h.sw50wupwtii0).

## 2.3 Phrase extraction

This will probably be the most complex and time-consuming components you’ll have to create for any Native SDK implementation. In order to extract the translatable strings from the codebase, you probably have two options:

* use the Abstract Syntax Tree (AST) feature of the language or of a 3rd-party library, if it exists.

* come up with the grammar that parses the programming language inner constructs you are interested in, i.e. "translate" function calls, i.e. t('This is a translatable string'), along with the supported metadata and other information.

For templates, the framework you are developing for will probably provide some kind of parsing functionality for its templating system you can take advantage of, like an AST structure or a visitor pattern that exposes the template tokens. Check out [Django’s templating system](https://docs.djangoproject.com/en/3.0/howto/custom-template-tags/#writing-custom-template-tags) as an example.

You will need to create the **Orchestrator** that detects and extracts translatable strings from template files and code files for a specific framework, and later feeds them to **CDSHandler**. For the moment, as there is no pushing functionality, the **Orchestrator** will simply extract the strings and stop there.

Phrase extraction (and later pushing) is something that doesn't need to be executed on runtime. Therefore, a typical implementation would be a CLI command that developers can run manually, or can integrate in a CI/CD tool to have it run automatically during deployments.

Since there are no files involved (Transifex Native works with a fileless approach, using payloads to CDS instead), extracting and pushing can be done in one step. For example, the related command for Django is ./manage.py transifex push, which extracts the strings and then pushes them to CDS.

### Expected result

When the Extractor is completed, you will be able to run a script that parses the codebase of a project and detects all translatable content, along with the supported metadata. You may implement both the programming language parsing (e.g. .py for Python) and the template code parsing (e.g. .html for web frameworks), or you can just implement one of them, skip to the pushing functionality and then come back for the other one.

### Metadata

Native SDKs should accept the following metadata for each translatable phrase:

#### Character limit

The maximum number of characters a translation for this string is allowed to have. Useful for making sure a text always fits in a UI element or a similar constraint. 

**Type:** `int`. Needs to be explicitly provided by the developer in the code/template.

#### Comment

A comment towards the translators. May contain valuable information for allowing them to deliver a better translation.

**Type:** `string`. Needs to be explicitly provided by the developer in the code/template.

#### Tags

Tags are used to categorize strings in arbitrary ways depending on the application, localization flow, etc, in order to achieve certain functionalities or aid in specific processes. Each string can have multiple tags.

**Type:** `Array[String]`. Needs to be explicitly provided by the developer in the code/template.
**Examples:** `['tag1', 'tag2']`

#### Occurrences

A translatable phrase might be found in multiple places in the same app. As long as the phrase and its (optional) context are identical, this will be treated as one phrase, with multiple instances. Having the information of where in the code a phrase appears can be convenient for many reasons, such as traceability. 

Each occurrence should be like <relative_file_path>:<line_of_code>, where relative means in relation to the app root folder (i.e. not the full absolute path of the file). 

**Type:** `Array[String]`. Should be automatically retrieved by the Native SDK during the string extraction phase.
**Example:** `['src/views/projects.py:137', 'src/api/strings.py:84]`

### Code examples

These are a few examples of translatable strings that the SDK should detect and extract. Depending on the programming language and framework the SDK is developed for, 

#### Django templates

```javascript
{% raw %}{% t "Red Five standing by." %}
{% t "Help me, {name}! You’re my only hope." _context="star wars" _charlimit=80 _comment="Very important; this is the first thing users see in the homepage" _tags="tag1,tag2" %}
{% t "{num, plural, one {There is {num} user in this team.} other {There are {num} users in this team.}}" num=team.users|length %}{% endraw %}
```

#### Python examples

```python
t("Welcome, {username}", username=user.name)

text = t(
    "{num, plural, "
    "    one {There is {num} user in this team.} "
    "    other {There are {num} users in this team.}"
    "}",
    num=total_users,
)
```

### Pseudocode

The following method should be called for each file that can host translatable strings. For example, in Django’s case this typically is .py, .html and .txt files. For other frameworks these would be different file types.

```javascript
function extract_strings_in_code(String src, String origin=null)
  // Parse the given code and extract translatable content.
  - src: a chunk of code
  - origin: the filename of the code this phrase appears in
  - return: a list of SourceString objects

  tree = ast.parse(src)
  visitor = CallDetectionVisitor(self._functions)
  visitor.visit(tree)
  source_strings, line_numbers = parse_source_strings(visitor.function_calls)
  // Detect occurrences
  for src_str, line_no in zip(source_strings, line_numbers):
    src_str.occurrences = ["{}:{}".format(origin, line_no)]
  return source_strings

// Simple struct object to hold information about each source phrase
class SourceString

  function init(String string, String context, Dictionary meta)
    this.key = generate_key(string, context)
    this.context_list = context.split(',')
    this.meta = meta

  property comment
    return this.meta['_comment']

  property character_limit
    return this.meta['_charlimit']

  property tags
    return this.meta['_tags']

  property occurrences
    return this.meta['_occurrences']
```

The implementation of CallDetectionVisitor and parse_source_strings is prohibitively big to write as pseudocode here, but you may check the [Python gettext implementation](https://github.com/transifex/transifex-python/blob/devel/transifex/native/parsing.py#L119) as well as the [Django implementation](https://github.com/transifex/transifex-python/blob/devel/transifex/native/django/management/utils/push.py#L149-L151) for inspiration.


## 2.4 Pushing source content
Create **CDSHandler** and specifically its pushing functionality. **CDSHandler** will use the [push API endpoint](#heading=h.kfa0dwq0y8j9) to send the data to CDS. 

Note that any requests to CDS should be [authenticated](#heading=h.9bq5qbsde8sw). In particular, pushing data requires both a *token* and a *secret*.

### Expected result

When this step is over you should be able to call a function (or a CLI command) that reads the project files, extracts the localized strings and sends them to CDS, which in turn sends them to Transifex, where they become available for translation.

### Pseudocode

```javascript
// Orchestrates the core functionality of the SDK
class NativeCore

  function init(Array[String] locale_codes, String token, String secret=null, String cds_host=null, Cache cache=null)
    this.locale_codes = locale_codes
    this.token = token
    this.secret = secret
    this.cds_handler = CDSHandler(locale_codes, token, secret, cds_host)
    this.cache = cache or MemoryCache()

  function push_strings(Array[SourceString] strings, bool purge=false)
    response = this.cds_handler.push_source_strings(strings, purge)
    // we can now show the results to the user
```

```javascript
// Takes care of all communication with CDS
class CDSHandler

  function init(Array[String] locale_codes, String token, String secret, String cds_host)
    this.locale_codes = locale_codes
    this.token = token
    this.secret = secret
    this.cds_host = cds_host

  // Push source strings to CDS.
  // - strings: a list of `SourceString` objects 
  // - purge: if true, any string that is not in `strings` will be deleted from CDS,
  //          if false all strings will be appended to those on CDS
  // - return: the HTTP response object
  function push_source_strings(strings, purge=false):

    if not this.secret:
      throw Exception(
        'You need to use `TRANSIFEX_SECRET` when pushing source content'
      )
    cds_url = this.cds_host + '/content/'

    // Construct `data` according to the expected payload of CDS,
    // using the properties and instance variables of SourceString
    data = {...}

    // Catch exceptions
    response = requests.post(
      self.host + cds_url,
      headers={
        'Authorization': 'Bearer {token}{secret}'.format(
          token=this.token,
          secret=':' + this.secret,
        ),
        'Accept-Encoding': 'gzip',
      }
      json={
        'data': data,
        'meta': {'purge': purge},
      }
    )

    return response
```

## 2.4 Pulling functionality

Implement the pulling functionality of **CDSHandler**, so that the **Cache** can be updated from actual content coming from CDS. You can ignore the **Daemon** for the time being and manually call the pull function.

Use the [pull API endpoint](#heading=h.wrun7eoearh).

### Expected result

When this step is over you should have an over-the-air (OTA) functionality that allows you to display new and modified translations to the user. Essentially you should have an end-to-end experience syncing source & translation content from and to Transifex.

Since the **Daemon** is not implemented yet, at the moment the translations will be fetched only once per entry point of the application. Depending on the type of the app, this could be:

* a page load for a frontend framework
* a server start of a backend framework
* an app launch for a mobile application

### Pseudocode

```javascript
class NativeCore

  // Fetch fresh content from the CDS.
  function fetch_translations()
    this.cache.update(this.cds_handler.fetch_translations())
```

```javascript
class CDSHandler

  function init(Array[String] locale_codes, String token, String secret, String cds_host)
    this.locale_codes = locale_codes
    this.token = token
    this.secret = secret
    this.cds_host = cds_host
    this.etags = {}

  // Fetch all translations for the given organization/project/(resource)
  // associated with the current token, for a specific language or all
  // supported languages
  // Returns a dictionary structured as
  // {
  //   '<locale_code>: {
  //     '<key>': {'string': '<translation>'},
  //     '<key>': {'string': '<translation>'},
  //     ...
  //   }
  // } 
  function fetch_translations(locale_code=null):

    cds_url = this.cds_host + '/content/{locale_code}'
    translations = {}
    if not locale_code
      languages = [lang['code'] for lang in this.fetch_languages()]
    else
      languages = [locale_code]

    // Get the intersection of the languages in the config and the languages
    // returned by CDS
    for locale_code in set(languages) + set(this.configured_locale_codes):
      try
        response = requests.get(
          this.host + cds_url.format(locale_code=locale_code),
          headers={
            'Authorization': 'Bearer {token}{secret}'.format(
              token=this.token,
              secret=':' + this.secret,
            ),
            'Accept-Encoding': 'gzip',
            'etag': etag=this.etags.get(locale_code),
          )
        )

        if not response.ok:
          raise Exception()

        // etag indicates that no translations have been updated
        if response.status_code == 304
          continue

        this.etags[locale_code] = response.headers.get('ETag', default='')
        translations[locale_code] = response.json()['data']
    catch
      logger.error('Error retrieving translations from CDS')

  return translations
```

```javascript
// Fetch the languages defined in the CDS for the specific project.
// Returned structure:
// {
//   "data": [
//     {
//       "name": "<lang name>",
//       "code": "<lang code>",
//       "localized_name": "<localized version of name>",
//       "rtl": <bool>
//     },
//     { ... }
//   ],
//   "meta": {
//     ...
//   }
// }
// - return: a dictionary of language information
function fetch_languages()
  cds_url = this.cds_host + '/languages'
  languages = []

  try
    response = requests.get(
      this.host + cds_url,
      headers={
        'Authorization': 'Bearer {token}{secret}'.format(
          token=this.token,
          secret=':' + this.secret,
        ),
        'Accept-Encoding': 'gzip', 
      }
    )
    if not response.ok:
      raise Exception()

    languages = response.json()['data']

  catch
      logger.error('Error retrieving languages from CDS')

  return languages
```

## 2.5 Daemon
Implement a mechanism that periodically retrieves new translations from CDS.

Depending on the type of application, it could make sense to retrieve the translations over the air periodically, without going through the normal entry point of the app. 

For example, if you are implementing Native for a backend framework, you wouldn’t want to have to restart the server in order to retrieve the translations. Instead you would like translations to be fetched on an interval, without restarting the server or doing a manual action to trigger this.

On the contrary, this wouldn’t make much sense for a mobile app or a frontend web application, as the entry point in these cases is accessed much more frequently through user-initiated action.

### Expected result
When this step is over you should have automatic OTA updates in your application in an interval-basis.

See an [example implementation](https://github.com/transifex/transifex-python/blob/master/transifex/native/daemon.py) in Python.

## 2.6 Additional features

### Policies

Implement **MissingPolicy** and **ErrorPolicy**. Start with pseudo-localization and a basic error policy, or create [additional ones](https://github.com/transifex/transifex-python/blob/master/transifex/native/rendering.py) if you feel like it. This step will probably require that you make changes in the signature of various functions throughout your codebase.

Apart from the implementation of these policies in the language core, you will probably also need to implement framework-specific ways to allow the developers to define the desired policies through project settings.

#### Missing Policy

Starting with Hello, friend as the source string, and assuming the translation in the target language (French) is missing, here are some other policies to consider implementing:

* `Hello, friend`
(source string; a sensible default option)
* `Ȟêĺĺø, ƒȓıêñđ`
([pseudo localization](https://en.wikipedia.org/wiki/Pseudolocalization); see [example character mapping](https://github.com/transifex/transifex-python/blob/master/transifex/native/rendering.py#L153-L171))
* `{{Hello, friend}}`
(source string wrapped in some kind of symbols; could be user-defined)
* `Hello, friend [fr]`
(source string followed by the target locale)
* `""`
(empty string)
* `missing translation missing translation miss…`
(a repeating string with hardcoded words that show the translation is missing; could be user-defined; its final length could be equal to the source string)
* `Hello, friend----`
(source string followed by extra characters to imitate locales that typically have longer words than English, such as German; extra length could be user-defined globally or per locale)

Another idea is to allow developers to define a composite policy that combines two or more policies together, as well as give them the ability to define their own policy.

The possibilities are endless; you may come up with other great policies that bring value to the user. 

##### Pseudocode

```javascript
// A missing policy that returns the source string followed by
// the target locale in between brackets.
class LocaleMissingPolicy

  function handle(String source_string, String locale_code)
    return source_string + ' [' + locale_code + ']'
```

```python
# Example
missing_policy = new LocaleMissingPolicy()
missing_policy.handle('Hello, friend', 'fr')
# returns 'Hello, friend [fr]'
```

#### Error Policy

##### Invalid translation

Let’s assume that `Hello, {friend}` is the source string and `Bonjour, {friend` is the  French translation (note that there is no closing brace). In ICU `{friend}` is a variable placeholder and it is expected to be replaced by an actual value during rendering.

The above translation has invalid ICU syntax, since there is no closing brace to mark the variable placeholder. This string cannot be properly rendered by an ICU renderer, so you need to handle it explicitly. Here are some other options to consider:

* `Hello, John`
(rendered source string)
* `Hello, {friend}`
(raw source string)
* `Bonjour, {friend`
(raw translation)
* `<!Hello, John!> or <!Hello, {friend}!> or <!Bonjour, {friend!>`
(rendered/raw source/translation, wrapped in some kind of symbols; could be user-defined)
* `[Invalid translation!]`
(hardcoded string; could be user-defined)
* A "decorator" policy that keeps track of the erroneous translations in a storage, so that SDK clients can do something custom with it (e.g. periodically check if there are erroneous strings and send an email to the localization team)

##### Invalid source string

If the source string also has invalid syntax, the options are fewer, as you cannot render it. These options aside, the rest is identical as above:

* `Hello, {friend}`
* `Bonjour, {friend`
* `<!Hello, John!> or <!Hello, {friend}!> or <!Bonjour, {friend!>`
* `[Invalid translation!]`
* A "decorator" policy that keeps track of the erroneous strings

##### Pseudocode

```javascript
// An error policy that tries to render the source string if possible,
// otherwise it falls back to returning the raw source string as is.
class RenderedSourceErrorPolicy

  function handle(STRING source_string, STRING translation, STRING locale_code, Dictionary params)
    // Try to render the source string, with the parameters (ICU variables) 
    try
      return StringRenderer.render(source_string, params)
    // If rendering fails, just return the raw source string
    catch ICURenderingException
      return source_string
```

```javascript
// A dummy implementation of the orchestrator entity NativeCore.
class NativeCore

  function translate(String source_string, String translation, String locale_code, Dictionary params) -> String
    try
      StringRenderer.render(translation, locale_code, params)
    catch
      error_policy = new RenderedSourceErrorPolicy()
      error_policy.handle(source_string, translation, locale_code)
```

```python
# Example with invalid translation, valid source string
NativeCore().translate('Hello, {friend}', 'Bonjour, {friend', 'fr', {friend='John'})
# returns 'Hello, John'

# Example with invalid translation, invalid source string
core.translate('Hello, {friend', 'Bonjour, {friend', 'fr', {friend='John'})
# returns 'Hello, {friend'
```

### Escaping

The `t` function should be changed in order to escape HTML text in order to avoid XSS attacks. A new function, `ut`, should be created in order to handle text that is considered safe, i.e. that should not be escaped.

You should probably expect to have to create a somewhat complex algorithm at the part where it is decided whether to escape the whole string (source or translation) and/or its variables, as well as some refactoring throughout the code.

*Note*: the Python SDK implements this business logic on [Django-specific code](https://github.com/transifex/transifex-python/blob/devel/transifex/native/django/templatetags/transifex.py#L192-L225), but ideally it would implement the core logic in a framework-agnostic module.

#### Expected result

When this step is over you support escaped and unescaped variants of the translate function.

```javascript
{% raw %}{% t "This is <b>nice</b>" %}{% endraw %}
# returns "This is &lt;b&gt;nice&lt;/b&gt;"

{% raw %}{% ut "This is <b>nice</b>" %}{% endraw %}
# returns This is <b>nice</b>
```

#### Pseudocode

```javascript
function t(String string, String context=null, Dictionary params) -> String {
  return TxNative.translate(
    sourceString: string,
    languageCode: NativeLocales.currentLocale,
    params: params,
    context: context,
    isSource: NativeLocales.isSource(NativeLocales.currentLocale),
    escape: false,
  )
}

# Identical to t(), but uses escape=True
function ut(String string, String context=null, Dictionary params) -> String {
  return TxNative.translate(
    sourceString: string,
    languageCode: NativeLocales.currentLocale,
    params: params,
    context: context,
    isSource: NativeLocales.isSource(NativeLocales.currentLocale),
    escape:true,
  )
}
```

```javascript
class TxNative

  function translate(String source_string, String locale_code, Dictionary params, String context=None, bool is_source=False, bool escape=True) -> String
    if is_source
      translation_template = source_string
    else
      key = generate_key(source_string, context)
      translation_template = this.cache.get(key, locale_code)

      if escape
        translation_template = html_escape(translation_template)
      // Parameters should be escaped anyway
      for key, value in params
        params[key] = html_escape(value)

    return render_string(source_string, translation_template, locale_code, params)
```

```javascript
function html_escape(String string)
  return string
    .replace('&', '&amp;')
    .replace('<', '&lt;')
    .replace('>', '&gt;')
    .replace('"', '&quot;')
    .replace("'", '&#039;')
```

You may check the [Django template tags](https://github.com/transifex/transifex-python/blob/053e036b472d272c62d26be8db84c36864e9702a/transifex/native/django/templatetags/transifex.py#L198-L204) for a more complex business logic of escaping parameters based on filters.

### Custom keys

Until now we’ve seen that the SDK generates keys automatically using the source string and its context. It is a good idea for the SDK to also support custom keys. This way, developers can define a humanly-readable key for a source string, which can make it easier to use them as IDs or search for them, etc.

#### Pseudocode

```javascript
// Orchestrates the core functionality of the SDK
class NativeCore

  function translate(String source_string, String locale_code, Dictionary params, String context=None, String key=null, bool is_source=False) -> String
    if is_source
      translation_template = source_string
    else
      key = key or generate_key(source_string, context)
      translation_template = this.cache.get(key, locale_code)

    return render_string(source_string, translation_template, locale_code, params)
```

#### Code examples

```javascript
{% raw %}{% t "Hello!" _key="welcome_msg" %}{% endraw %}
t("Hello", _key="welcome_msg")
```

### Proper plural handling

#### Key generation

The algorithm that generates keys and supports pluralized strings is quite complex. The reason is that the content for each plural rule must be extracted separately. For an ICU string that looks like {% raw %}`{cnt, plural, one {A table} other {{cnt} tables}}`{% endraw %}, the following intermediate string must be created: `1:A table:5:{cnt} tables`. 

This means that the original string must be parsed using the ICU grammar and return all necessary components. The **ICUMessageFormat** library you have integrated in or built for the SDK may already have this functionality, in which case key generation will be straightforward.

##### Pseudocode

The high level algorithm would look like this:

```javascript
// Return a unique key based on the given source string and optional context.
// A string can be associated with multiple context values, so the context
// argument can be a serialized comma-separated string or a single string.
// Support ICU strings or pre-parsed dictionaries with all plurals,
// source_string: an ICU string
// - context: an optional context, or multiple comma-separated contexts
// - plurals: a dictionary with pre-parsed pluralized content, e.g.
//            {1: "This is a dog", 5: "These are dogs"}
function generate_key(String source_string=null, String context=null, Dictionary<int, String> plurals=null) -> String

  if source_string and plurals
    raise Exception('Cannot provide both `source_string' and `plurals`)

  if not source_string and not plurals
    raise Exception('Need to provide either `source_string' or `plurals`)

  if not plurals
    is_pluralized, plurals = parse_plurals(source_string)
    // returned plurals should look like {1: "This is a dog", 5: "These are dogs"}

  items = []
  // Always sort to ensure the key stays the same even if
  // the order of plurals inside the ICU string changes
  for plural_index, plural_string in plurals.sort_by_key()
    items.push(plural_index + ':' + plural_string)

  string_content = items.join(':')
  if context
    context = context.replace(',', ':')

  if not context
    context = ''

  return md5(string_content + ':' + context)
```

You’ll notice that the implementation of parse_plurals() is omitted from the above pseudocode. The reason is that it's too complex to provide in full. If you’d like a full-blown implementation, you can check a [Python example](https://github.com/transifex/transifex-python/blob/053e036b472d272c62d26be8db84c36864e9702a/transifex/common/utils.py#L95-L225) (this particular one uses direct algorithmic parsing instead of an expression grammar).

#### Rendering pluralized strings

So far, we have been treating the translations that come from CDS as final. However, the translations of pluralized strings need special handling.

If a string is pluralized, its translation that arrives to the SDK will look like {% raw %}`{???, plural, one {A table} other {{cnt} tables}}`{% endraw %}. In order to create the correct translation template, we have to replace `???` with the actual count variable that is found in the source string, which in the case above is cnt.

```javascript
class TxNative

  function translate(String source_string, String locale_code, Dictionary params, String context=None, bool is_source=False, bool escape=True) -> String
    if is_source
      translation_template = source_string
    else
      key = generate_key(source_string, context)
      translation_template = this.cache.get(key, locale_code)
      is_pluralized, plurals = parse_plurals(translation_template)

      if is_pluralized and translation_template.startswith('{???')
        count_var_name = source_string.substring(1, source_string.indexof(',')).trim()
        translation_template = translation_template.replace('???', count_var_name, limit=1)

      if escape
        translation_template = html_escape(translation_template)

      // Parameters should be escaped anyway
      for key, value in params
        params[key] = html_escape(value)

    return render_string(source_string, translation_template, locale_code, params)
```

### Lazy implementation

This is a nice-to-have feature and its applicability depends on the programming language, the framework and the type of apps it is used for (e.g. frontend, backend, etc).

There might be cases where a string is marked for localization in a place in the code that is executed before the Native SDK had a chance to fetch the translations or even before it had a chance to be initialized.

For these cases, a lazy implementation of the t and ut functions comes in handy. Check the following Python example:

```python
class MyView(TemplateView):

    description = t('The description')

    def get_context_data(self, **kwargs):
        return {'msg': t('The message')}
```

What Python does when reading the source .py file above, is to go through all lines of code. While the t() function in 'msg': t('The message') will only be executed when get_context_data() is actually called during the handling of an HTTP request, description = t('The description') will be called immediately the first time this file (module) is read by the interpreter. 

The problem is that this happens very early in the program execution, and at that time the Native SDK hasn’t been initialized yet. Even if it had, the translations will certainly not be in yet. The solution is to implement a lazy variant:

```python
class MyView(TemplateView):

    description = lazyt('The description')
```

Now the evaluation, i.e. the rendering of the string, will be done at the very last moment, at which point the translations should have arrived, just like with any other example we’ve seen so far.

You can check out an [implementation example](https://github.com/transifex/transifex-python/blob/053e036b472d272c62d26be8db84c36864e9702a/transifex/common/strings.py#L108-L232) in Python.

## 2.7 Configuration

These are the suggested configuration options to provide to the developers that integrate the Native SDK:

#### Token & secret [required]

The token and secret used for authenticating with CDS.

```javascript
TRANSIFEX_TOKEN = 'xxxxxx'
TRANSIFEX_SECRET = 'xxxxxx'
```

For applications whose codebase is publicly exposed to the end user, such as frontend apps, the secret should not be added inside the code, as that would be a security issue.

However, the strings need to be pushed to CDS, so the secret is required. For these cases, you could consider creating a CLI entry point that requires a value for TRANSIFEX_SECRET and uses it in order to push the source strings to CDS via **CDSHelper**. This way, developers will be able to take advantage of the SDK in order to push the strings, by providing the secret every time as a command argument, or as an environmental variable, without exposing it to the end users.

```bash
# Secret is internally read as an environment variable
$ push-strings

# Alternatively, secret is provided as an argument
$ push-strings -secret='xxxxxx'
```

#### Languages [required]

A list of locales that the application should support, and the application’s source language.

```javascript
LANGUAGES = ['fr', 'de_DE', 'pt_BR']
SOURCE_LANGUAGE = 'en'
```

#### CDS host [optional]

The host of the CDS service. If not provided, the Native SDK could fallback to https://cds.svc.transifex.net.

```javascript
TRANSIFEX_CDS_HOST = 'https://cds.myservice.com'
```

#### Missing Policy [optional]

The policy to use when a translation is missing.

Ideally this would be in a format that would allow developers to define their own custom implementation, in addition to any existing policies provided by the SDK. An example would be a module path in Python or a direct instance reference if the framework allows it.

A sensible fallback is a policy that returns the rendered source string.

```javascript
TRANSIFEX_MISSING_POLICY = 'native.policies.missing.SourceStringPolicy'
```

#### Error Policy [optional]

The policy to use when a translation or its source string are invalid. Ideally this would be in a format that would allow developers to define their own custom implementation, in addition to any existing policies provided by the SDK.

A sensible fallback could be a policy that returns a hardcoded string when a translation is invalid, and another string when the source string is invalid.

```javascript
TRANSIFEX_ERROR_POLICY = 'native.policies.error.SourceStringPolicy'
```

#### Sync interval [optional]

Only makes sense if the **Daemon** is implemented. Determines how often the SDK will request fresh translations from CDS.

```javascript
TRANSIFEX_SYNC_INTERVAL = 60 * 60 // 60 minutes
```

Previous: [Core components](core_components.md)  |  Next: [Content Delivery Service](cds.md)
