*(Estimated time: 15mn)*

Your current Sugarizer session talks probably the same language as you. At the first setup, Sugarizer detects the language of your browser and uses this language for the UI and the activities

You could also change the language from the settings. Hover the mouse on the XO buddy icon on the Sugarizer home view and then click on settings, then to "Language" to display the language settings window.


![](images/tutorial_step5_1.png)

If you choose another language, this new language will be used for all activities. Let's see how we could use it in our Pawn activity too.

# Identify strings to localize

Sugarizer and Sugar-Web use the [webL10n](https://github.com/fabi1cazenave/webL10n) JavaScript library to handle localization.

The first step when you localize an activity is to identify strings to localize. It means replace hard-coded string in your HTML or JavaScript files by localization resources. In the Pawn activity we've got three strings to localize:

* *"Hello {user}"*: the welcome message
* *"{user} played!"*: when the user played a pawn
* *"Add pawn"*: the helper message on the toolbar button

With the webL10n library, all strings have to be defined in a specific file where all translations for each string should be set. We call this file `locale.ini`.  So using your text editor, let's create a `locale.ini` file at the root of the Pawn activity. Here what it looks likes: 
```ini
[*]
Hello=Hello {{name}}!
Played={{name}} played
AddPawn=Add pawn

[en]
Hello=Hello {{name}}!
Played={{name}} played
AddPawn=Add pawn

[fr]
Hello=Bonjour {{name}} !
Played={{name}} a joué
AddPawn=Ajouter pion

[es]
Hello=Hola {{name}} !
Played={{name}} jugó
AddPawn=Agrega un peón
```

This file is decomposed into sections. One section for each language with the language code between brackets as section name (**en**, **fr**, **es**, ...) and a special section (**\***) for unknown language.

In each section, you have to define translations for each string. The left side of the equal sign is the id of the string (**Hello**, **Played**, **AddPawn**), the right side of the equal sign is the translated string.

For parameterized strings (i.e. strings where a value is inside the string), the double curved bracket **\{\{\}\}** notation is used.

Now we will integrate our new `locale.ini` file into `index.html`:
```html
<!DOCTYPE html>
<html>

<head>
	<meta charset="utf-8" />
	<title>Pawn Activity</title>
	<meta name="viewport"
		content="user-scalable=no, initial-scale=1, maximum-scale=1, minimum-scale=1, width=device-width, viewport-fit=cover" />

  <!-- Add locale.ini file -->
  <link rel="prefetch" type="application/l10n" href="locale.ini">

	<link rel="stylesheet" media="not screen and (device-width: 1200px) and (device-height: 900px)"
		href="lib/sugar-web/graphics/css/sugar-96dpi.css">
	...
```

The file is included as a HTML `link` tag and must have a `application/l10n` type to be recognized by webL10n library.

# Initialize localization

We will now see how to initialize localization into the activity source code.

Once again we will have first to integrate a new component. So let's add the `SugarL10n` to `index.html` and create an instance:
```html
    ...
    <sugar-journal ref="SugarJournal" v-on:journal-data-loaded="onJournalDataLoaded"></sugar-journal>
    <sugar-localization ref="SugarLocalization"></sugar-localization>
  </div>

  <script src="js/Pawn.js"></script>
  <script src="js/components/SugarActivity.js"></script>
  <script src="js/components/SugarToolbar.js"></script>
  <script src="js/components/SugarJournal.js"></script>
  <script src="js/components/SugarL10n.js"></script>
  <script src="js/activity.js"></script>
</body>
```

Now in `js/activity.js`, let's keep a data variable as a reference to this component instance as it might be used multiple times in the activity.
```js
data: {
  currentenv: null,
  SugarLocalization: null,
  ...
},
mounted: function () {
  this.SugarLocalization = this.$refs.SugarLocalization;
},
```

The `SugarL10n` component automatically detects the language set by the user using the environment and configures the webL10n library accordingly.

# Set strings value depending on the language

To get the localized version of a string, the webL10n framework provide a simple `get` method. You pass to the method the id of the string (the left side of the plus sign in the INI file) and, if need, the string parameter. You can call the `get` method of the Sugar component which will work the same way. The `SugarL10n` also provides a `localize(object)` method to localize a JavaScript object containing string id's. 

So for the welcome message, here is the line to write:
```js
this.displayText = this.SugarLocalization.get("Hello", { name: this.currentenv.user.name });
```
As you could see the first `get` parameter is the id of the string (**Hello**) and the second parameter is a JavaScript object where each property is the name of the parameter (the one provided inside double curved brackets **\{\{\}\}**, **name** here). The result of the function is the string localized in the current language set in webL10n.

In a same way, the pawn played message could be rewrite as: 
```js
this.displayText = this.SugarLocalization.get("Played", { name: this.currentenv.user.name });
```

To set localized titles to toolbar items, let's define a JavaScript object `l10n` which will store string id's of the strings we want (strings here should be static, i.e. without parameters)
```js
data: {
  ...
  l10n: {
    stringAddPawn: ''
  }
}
```
**NOTE:** *The string id's inside the object should be prefixed by the word "string". For example in this case we want the string for `AddPawn`, so in the object we will write `stringAddPawn: ''`.*

We can localize this object by calling the `localize` method: 
```js
this.SugarLocalization.localize(this.l10n);
```

Let's bind this to the title of add-button:
```html
<sugar-toolitem id="add-button" v-bind:title="l10n.stringAddPawn" v-on:click="onAddClick"></sugar-toolitem>
```

One point however: we need to wait to initialize strings that the `locale.ini` is read. It's possible because the webL10n framework raises a new `localized` event on the window when the language is ready. Our `SugarL10n` component will emit a Vue event called `localized` as well.

So we will now initialize the welcome message in the `localized` event listener, which we add in `js/activity.js` file:
```js
initializeActivity: function () {
  // Initialize Sugarizer
  this.$refs.SugarActivity.setup();
  this.currentenv = this.$refs.SugarActivity.getEnvironment();

  /*Set the event listener here */
  this.SugarLocalization.$on('localized', this.localized());
},

// Handles localized event
localized: function () {
  this.displayText = this.SugarLocalization.get("Hello", { name: this.currentenv.user.name });
  this.SugarLocalization.localize(this.l10n);
},
```
**NOTE:** *We define the event listener using `$on` in `initializeActivity()` rather than as a `v-on` directive on the `<sugar-localized>` tag to maintain the flow of the activity. Defining the event listener as a directive might cause the `localized()` method to be called BEFORE the `currentenv` is set. This will lead to undesired results.*

Everything is now ready to handle localization.

Let's test it. Change the Sugarizer language to French and let's see the result.


![](images/tutorial_step5_2.png)

The welcome message and the button placeholder is now in French. The played message works too.