# How to theme a Flutter app

It's not the only article of its kind, far from it. Still, most of what's availabe on the net simply falls short of the 
requirements. Either it doesn't offer an easy way for the user to change themes, or it doesn't follow the system theme 
changes (maybe it does, after an app restart, but not immediately while running), or doesn't include the system 
navigation elements, or shows an unsightly flicker during app startup. Or, if it does most everything correctly, it 
doesn't allow for simple, easy specification of the mai and accent colors.

So, let me introduce my solution that I built up during many years of flutter development and now I copy from app to app 
when I need to start a new one. It will span a few classes for sure, still, I think, when all is coming together, it's 
relative simple for what it can accomplish and it really offers all necessary features while still remains 
straightforward to customize.

In the background, this approach doesn't introduce any new concepts, it just uses the underlying theming mechanism 
Flutter offers. Most importantly, it relies on a provider to inject the necessary information into the app build 
process. So, let's introduce this provider first,

## ThemeManager

I call it `ThemeManager`. It simply manages the theme related settings, stored in shared preferences. It is a 
`ChangeNotifier` so that it can notify interested parties (basically, the app itself) of any changes to the required 
theme.

```dart
import 'dart:ui' as ui;

import 'package:flutter/material.dart';
import 'package:flutter/services.dart';
import 'package:shared_preferences/shared_preferences.dart';

class ThemeManager with ChangeNotifier {
  final SharedPreferences prefs;
  var brightness = Brightness.light;
  String _theme = 'system';

  ThemeManager(this.prefs) {
    brightness = ui.PlatformDispatcher.instance.platformBrightness;
    _theme = prefs.getString('theme') ?? 'system';
    notifyListeners();
  }

  String get theme => _theme;

  set theme(String value) {
    prefs.setString('theme', _theme = value);
    notifyListeners();
  }

  ThemeMode get themeMode => switch (_theme) {
        'light' => ThemeMode.light,
        'dark' => ThemeMode.dark,
        'system' => (brightness == Brightness.light) ? ThemeMode.light : ThemeMode.dark,
        _ => ThemeMode.light,
      };

  SystemUiOverlayStyle get systemStyle => (themeMode == ThemeMode.light) //
      ? SystemUiOverlayStyle.light.copyWith(
          statusBarColor: Colors.white,
          systemNavigationBarColor: Colors.white,
          statusBarIconBrightness: Brightness.light,
          systemNavigationBarIconBrightness: Brightness.light,
        )
      : SystemUiOverlayStyle.dark.copyWith(
          statusBarColor: Colors.black,
          systemNavigationBarColor: Colors.black,
          statusBarIconBrightness: Brightness.dark,
          systemNavigationBarIconBrightness: Brightness.dark,
        );
}
```

Note that it gets the `SharedPreferences` instance from outside. This might seem odd, why it doesn't read the instance 
itself, setting the variable in a `then()` callback and notifies the listeners then? That would be the proper way, 
wouldn't it?

Well, maybe, from a purist point of view, yes. But doing so would mean a flicker during app startup if the system is, 
for instance, light and the app is set to dark. It would start in light mode and change to dark immediately as the 
`ThemeManager` reads the proper theme in an async way. We have to avoid this.

## The app

So, let's examine the app itself. It will be more or less the same as any usual Flutter app, but with a couple subtle 
differences. First, we will have an async `main()`. Note that we need to ensure the proper initialization of the
Flutter bindings before we can use anything async:

```dart
void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  runApp(MyApp(prefs: await SharedPreferences.getInstance()));
}
```

This is exactly where we get the preferences in a syncish way and only start the app when we already have it. Fortunately,
even if it's technically async, it's rather fast.

Second, our app will be stateful, not stateless, as most examples and samples would show:

```dart
class MyApp extends StatefulWidget {
  final SharedPreferences prefs;

  const MyApp({super.key, required this.prefs});

  @override
  State<MyApp> createState() => _MyAppState();
}
```

Going stateful has a very important reason, actually: we have to observe changes in the system theme so that our app can 
act upon it, changing its own theme immediately, without needing to restart first:

```dart
class _MyAppState extends State<MyApp> with WidgetsBindingObserver {
  var refreshKey = UniqueKey();

  @override
  void initState() {
    super.initState();
    WidgetsBinding.instance.addObserver(this);
  }

  @override
  void dispose() {
    WidgetsBinding.instance.removeObserver(this);
    super.dispose();
  }

  @override
  void didChangePlatformBrightness() {
    super.didChangePlatformBrightness();
    setState(() => refreshKey = UniqueKey());
  }
```

To accomplish that, we need a `WidgetsBindingObserver` that we can subscribe to and ask for notifications when the 
platform brightess (aka theme) changes. And we rely on the fact that Flutter will rebuild widgets (even the full widget 
tree in this case) when it has a uniquely keyed widget that changes. So, we simply replace the key with a new one to 
force a rebuild. Some might call this a hack but it's actually a widely used mechanism in Flutter.

```dart
  @override
  Widget build(BuildContext context) => MultiProvider(
        key: refreshKey,
        providers: [
          ChangeNotifierProvider<ThemeManager>(create: (_) => ThemeManager(widget.prefs)),
        ],
        child: Consumer<ThemeManager>(builder: (context, themeManager, child) {
          return buildApp(themeManager);
        }),
      );
```

Anyway, our `build()` will provide the theme manager, keyed with this unique key, to our app:

```dart
  Widget buildApp(ThemeManager theming) {
    return AnnotatedRegion(
      value: theming.systemStyle,
      child: MaterialApp(
        themeMode: theming.themeMode,
        theme: AppTheme.light,
        darkTheme: AppTheme.dark,
        // ...
        home: const MainPage(),
      ),
    );
  }
}
```

We have four points of interest here: we supply the actual theme from the `ThemeManager`, as well as the system styles 
(status and navigation bars). And we also provide our actual themes for the system to chose from: two `AppTheme` variants.

## The actual theming

This is where the fun begins. First, we set up two base themes, one light and one dark, providing our main input: the 
seed color. Flutter has a nice way to create a whole color palette from just a single color, creating all variations, 
lighter and darker versions, contrasting colors, everything. You might also specify the primary, or the secondary, or 
even the tertiary color but my recommendation would be to provide as little as possible. All apps differ, of course, but 
my usual way is to give a primary because I often want to have a specific one rather than the one generated from the 
seed, and maybe a secondary if the design calls for a specific secondary accents. I prefer to leave everything else to 
the underlying platform code because if you start to go any deeper, you usually open a can of worms and some 
combinations will not be contrasting or pleasing enough. But it's up to you, of course.


```dart
class AppTheme {
  static var lightBase = ColorScheme.fromSeed(
    brightness: Brightness.light,
    seedColor: Colors.red,
    primary: Colors.red,
    secondary: Colors.orange.shade600,
    background: Colors.white,
  );
  static var darkBase = ColorScheme.fromSeed(
    brightness: Brightness.dark,
    seedColor: Colors.red,
    primary: Colors.red,
    secondary: Colors.orange.shade600,
    background: Colors.black,
  );

  AppTheme._();
```

This is just the starting point. The next item will be a helper function that starts with a scheme and provides the 
actual theming for the various elements of the design. The next bock is just an arbitrary sample from one of my apps. 
`ThemeData` has a huge list of supported fields, one for every conceivable UI control. The same applies here: use is 
sparingly. Always check what the default offers and only specify your own theme if you need to deviate from the default. 
And even where you do, try to always use existing colors from the `scheme` rather than providing your own, ad hoc colors.

```dart
  static ThemeData fullTheme(Brightness brightness, ColorScheme scheme) => ThemeData(
        useMaterial3: true,
        brightness: brightness,
        visualDensity: VisualDensity.adaptivePlatformDensity,
        colorScheme: scheme,
        appBarTheme: AppBarTheme(
          color: scheme.primary,
          foregroundColor: scheme.onPrimary,
          iconTheme: IconThemeData(color: scheme.onPrimary),
        ),
        navigationBarTheme: NavigationBarThemeData(
          backgroundColor: scheme.primaryContainer,
          indicatorColor: scheme.secondaryContainer,
          iconTheme: MaterialStateProperty.all(IconThemeData(color: scheme.onPrimaryContainer)),
          labelTextStyle: MaterialStateProperty.all(const TextStyle()),
        ),
        bottomAppBarTheme: BottomAppBarTheme(
          color: scheme.primaryContainer,
        ),
        floatingActionButtonTheme: FloatingActionButtonThemeData(
          backgroundColor: scheme.secondary,
          foregroundColor: scheme.onSecondary,
        ),
        filledButtonTheme: FilledButtonThemeData(
          style: FilledButton.styleFrom(backgroundColor: scheme.primaryContainer, foregroundColor: scheme.onPrimaryContainer),
        ),
        textTheme: TextTheme(
          titleSmall: TextStyle(color: scheme.primary),
          titleMedium: TextStyle(color: scheme.onBackground, fontWeight: FontWeight.normal),
          titleLarge: TextStyle(color: scheme.onBackground),
        ),
        cupertinoOverrideTheme: macTheme(brightness, scheme),
      );

  static CupertinoThemeData macTheme(Brightness brightness, ColorScheme scheme) => CupertinoThemeData(
        brightness: brightness,
        primaryColor: scheme.primary,
        barBackgroundColor: scheme.surface,
        primaryContrastingColor: scheme.onPrimary,
        textTheme: CupertinoTextThemeData(
          primaryColor: scheme.primary,
          actionTextStyle: TextStyle(color: scheme.onPrimary),
          navTitleTextStyle: TextStyle(color: scheme.onPrimary),
          tabLabelTextStyle: TextStyle(color: scheme.onPrimary),
        ),
      );

  static final ThemeData light = fullTheme(Brightness.light, lightBase);
  static final ThemeData dark = fullTheme(Brightness.dark, darkBase);
}
```

Basically, that's it. You have a themed app that gets its color palette automatically from a few provided base colors, 
spreads its centralized theming to all the widgets used throughout the app to ensure a consistent look. It reacts to the 
system theme change automatically, on the fly. It starts perfectly, without flickering. The only element still missig is
a way for the user to change the theming at will.

## User settings

Creating a settings page is beyond the scope of this article—you'll find plenty of examples on the web everywhere. But once you have one,
be sure to include a selection for the user to select from three options:

```dart
  Map<String, String> _THEMES(BuildContext context) {
    return {
      'system': 'system',
      'light': 'Light',
      'dark': 'Dark',
    };
  }
```

You can use a `SimpleDialog` that has three `SimpleDialogOption` children for these items and when the user has selected one,
push the selected one into the `ThemeManager` passed along to your settings page:

```dart
class SettingsPage extends StatefulWidget {
  final ThemeManager theming;

  const SettingsPage({super.key, required this.theming});

  @override
  State<SettingsPage> createState() => _SettingsPageState();
}
```

with a simple assignment:

```dart
widget.theming.theme = selectedTheme;
```

As the manager both stores the selection and notifies its listeners, the app theme will change immediately.
