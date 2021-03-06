# angular-localize

> A localization module for [AngularJS](http://angularjs.org/).

## Table of contents

- [Getting started](#getting-started)
    - [Module setup](#module-setup)
    - [Translation functions](#translation-functions)
        - [How to automatically create the translation functions](#how-to-automatically-create-the-translation-functions)
- [Usage Examples](#usage-examples)
    - [localize directive](#localize-directive)
        - [Localize using the element content](#localize-using-the-element-content)
        - [Localize using the localize attribute](#localize-using-the-localize-attribute)
        - [Localize with dynamic user data](#localize-with-dynamic-user-data)
        - [Configuring the observable directive attributes](#configuring-the-observable-directive-attributes)
    - [localize service](#localize-service)
    - [Localize filter](#localize-filter)
    - [Localize factory](#localize-factory)
- [License](#license)

## Getting started

### Module setup
The easiest way to install the `localize` module is via [Bower](http://bower.io/):

```shell
bower install angular-localize --save
```

You can then include `angular-localize` after including its dependencies, [angular](https://github.com/angular/bower-angular) and [angular-sanitize](https://github.com/angular/bower-angular-sanitize):

```html
<script src="bower_components/angular/angular.js"></script>
<script src="bower_components/angular-sanitize/angular-sanitize.js"></script>
<script src="bower_components/angular-localize/angular-localize.js"></script>
```

### Translation functions
The `localize` module requires a map with translation functions.  
By convention, this is a global object variable called `i18n`, which must be available before the Angular application is initialized:

```js
window.i18n = {
    'Hello {name}!': function (data) {
        return 'Hallo ' + data.name + '!';
    }
};
```

The `localize` module uses this map to lookup the translation results.  
If no matching translation function is found, the key is used as the translation result.

Instead of storing the translation functions in a global object, it's also possible to decorate the `localizeConfig` service to override the `i18n` configuration property:

```js
angular.module('localize').config(['$provide', function ($provide) {
    $provide.decorator('localizeConfig', ['$delegate', function ($delegate) {
        $delegate.i18n = {
            Apples: function () {
                return 'Äpfel';
            },
            'Hello {name}!': function (data) {
                return 'Hallo ' + data.name + '!';
            }
        };
        return $delegate;
    }]);
}]);
```

The translation functions are expected to return strings with the translation result.  
An optional object with dynamic user data is passed as only argument to the translation functions.

#### How to automatically create the translation functions
[grunt-locales](https://github.com/blueimp/grunt-locales), a plugin for the [Grunt](http://gruntjs.com/) task runner, provides command-line scripts to automate the creation of the translation functions.

[grunt-locales](https://github.com/blueimp/grunt-locales) parses `localize` attributes in HTML files as well as `localize` method calls in JS files and collects the parsed locale strings in JSON files for translation.  
The translated JSON locale files are then compiled into JavaScript files containing the map of translation functions.

To support translation features like pluralization and gender selection, [grunt-locales](https://github.com/blueimp/grunt-locales) relies on Alex Sexton's [MessageFormat](https://github.com/SlexAxton/messageformat.js) library to parse the locale strings and compile the translation functions.

## Usage Examples

### localize directive

#### Localize using the element content
Any HTML element which can contain text nodes can be localized simply by adding the `localize` attribute:

```html
<p localize>Save the Orangutans!</p>
```

If a translation function for the key `"Save the Orangutans!"` exists, the `localize` directive will replace the element content with the result of executing the function.

Localized element content can also contain HTML:

```html
<p localize>Save the <strong>Orangutans</strong>!</p>
```

In this case, the key for the translation function is `"Save the <strong>Orangutans</strong>!"`.

The result of the translation function of localizations defined via element content will always be assigned as HTML, but sanitized via [angular-sanitize](https://github.com/angular/bower-angular-sanitize).

#### Localize using the localize attribute
Instead of the element content, the localization key can also be defined as value of the `localize` attribute:

```html
<p localize="Save the Orangutans!"></p>
```

If no translation function for the key `"Save the Orangutans!"` exists, the attribute value will be used as element content.

Localizations defined via `localize` attribute cannot contain HTML tags, as the translation result will be assigned as text, not as HTML. This limitation enables a slightly faster localization, as no sanitization is required.

#### Localize with dynamic user data
It's also possible to provide dynamic user data to the translation functions.

The `localize` directive observes all non-directive `data-*` attributes and passes them as normalized map of key/value pairs to the translation function:

```html
<p data-name="{{user.name}}" localize="Hello {name}!"></p>
```

Whenever `user.name` is updated, the translation function for `"Hello {name}!"` gets called with an object, e.g. `{name: 'Bob'}` as argument and the element content is updated accordingly.

This also works with the localization key as element content, which allows the use of HTML for the translation result:

```html
<p data-name="{{user.name}}" localize>Hello <strong>{name}</strong>!</p>
```

In this case, all dynamic user data is escaped (HTML special characters are replaced with their respective HTML entity) before it is passed to the translation function.

#### Configuring the observable directive attributes
By default, the `localize` directive only observes non-directive data-attributes, which are identified by the `observableAttrs` regular expression, a property of the `localizeConfig` service.  
You can override the default setting by decorating the `localizeConfig` service:

```js
angular.module('localize').config(['$provide', function ($provide) {
    $provide.decorator('localizeConfig', ['$delegate', function ($delegate) {
        $delegate.observableAttrs = /^data-(?!ng-|localize)/;
        return $delegate;
    }]);
}]);
```

The regular expression is matched against the non-normalized attribute names of the directive. The default setting will match any `data-*` attributes which are not followed by the strings `ng-` or `localize`.

### localize service
The `localize` service is an equivalent to the `localize` directive and can be used to generate localized results in situations where the directive cannot be used:

```js
angular.module('example')
    .controller([
        '$scope', 'localize',
        function ($scope, localize) {
            $scope.text = localize(
                'Hello {name}!',
                {name: $scope.user.name}
            );
        }
    ]);
```

The `localize` service expects the localization key as first argument and an optional object with user data as second argument.

If the third argument is set to `true`, the user data will be escaped (HTML special characters are replaced with their respective HTML entity), which allows to output the translation result as HTML, although it still needs to be properly sanitized depending on the security context:

```js
angular.module('example')
    .controller([
        '$scope', 'localize',
        function ($scope, localize) {
            $scope.text = localize(
                'Hello <strong>{name}</strong>!',
                {name: $scope.user.name},
                true
            );
        }
    ]);
```

Generally, it is preferable to use the `localize` directive instead of the service whenever possible, as the directive can determine its security context.  
However, [grunt-locales](https://github.com/blueimp/grunt-locales) is able to parse both the directive attributes as well as the localize service calls.

### localize filter
The `localize` filter provides the same functionality as the service.  
It can be useful in templates where the localization strings are dynamic, e.g. for error messages:

```html
<p>{{errorMessage | localize}}</p>
```

It is also possible to pass an object with localization arguments to the `localize` filter:

```html
<p>{{errorMessage | localize:data}}</p>
```

If the result of the localization is rendered as HTML, the localization arguments must be escaped by setting the second filter argument to `true`:

```html
<p ng-bind-html="errorMessage | localize:data:true"></p>
```

### localize factory
The `localizeFactory` is a service to facilitate the creation of additional, attribute-based `localize` directives.

Executing the `localizeFactory` function returns a [directive](http://docs.angularjs.org/guide/directive) definition object that identifies the attribute to update via its directive name.

The following is the simple code required to create a `localize-title` directive:

```js
angular.module('example')
    .directive('localizeTitle', ['localizeFactory', function (localizeFactory) {
        return localizeFactory();
    }]);
```

Directives created via the `localizeFactory` work the same way as the original `localize` directive, but instead of updating the element content, they update their associated HTML attribute.  

The following is an example use of the created `localize-title` directive:

```html
<p data-name="{{user.name}}" localize-title="Hello {name}!"></p>
```

This directive will update the `title` attribute based on the translation function and provided user data.

The updated attribute is always the directive name with the `localize-` prefix removed.  
Therefore, as a convention, the directive name must always start with `localize` as prefix.

## License
Released under the [MIT license](http://www.opensource.org/licenses/MIT).
