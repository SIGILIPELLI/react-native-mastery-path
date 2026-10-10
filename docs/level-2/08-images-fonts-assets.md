---
description: "Handle images with expo-image caching and placeholders, ship @2x/@3x assets, load custom fonts behind the splash screen, and add vector icons."
---

# 08 · Images, Fonts & Assets

Assets are where "it looked fine on my phone" goes to die: a 4 MB photo decoded into a 100-point
thumbnail, a list that re-downloads every avatar on scroll, a custom font that flashes in after
the system font, an icon that's blurry on a 3× screen. This lesson covers loading images
efficiently, bundling resolution variants, fonts, icons and the splash screen.

## Bundled images and resolution variants

Local images are imported with `require`, and Metro picks the right density variant automatically:

```
assets/
├── empty-state.png      ← 1× (e.g. 200×160 px)
├── empty-state@2x.png   ← 2× (400×320 px)
└── empty-state@3x.png   ← 3× (600×480 px)
```

```tsx
<Image source={require('../assets/empty-state.png')} />
```

The component is laid out at the 1× size (200×160 points), and on a 3× phone the `@3x` file is used,
so it's crisp. You don't need to set `width`/`height` for bundled images — Metro records their
dimensions at build time. (`require` paths must be **static strings**; Metro resolves them when it
bundles, so `require(\`./icons/${name}.png\`)` doesn't work. Build a map of requires instead.)

```ts
const moodImages = {
  happy: require('../assets/mood-happy.png'),
  meh: require('../assets/mood-meh.png'),
  sad: require('../assets/mood-sad.png'),
} as const;
```

## expo-image for remote images

The core `Image` works, but for real apps `expo-image` adds what you'll otherwise build yourself:
memory and disk caching, placeholders (including compact **BlurHash**/**ThumbHash** strings),
cross-fade transitions, and better format support.

```bash
npx expo install expo-image
```

```tsx
import { Image } from 'expo-image';

<Image
  source={{ uri: recipe.thumbnail }}
  style={{ width: '100%', aspectRatio: 4 / 3, borderRadius: 12 }}
  contentFit="cover"
  placeholder={{ blurhash: 'L6PZfSi_.AyE_3t7t7R**0o#DgR4' }}
  transition={200}
  cachePolicy="memory-disk"
  accessibilityLabel={recipe.name}
/>
```

- `contentFit` replaces `resizeMode` (`cover`, `contain`, `fill`, `none`, `scale-down` — the CSS
  `object-fit` names).
- `placeholder` shows instantly while the real image loads; a BlurHash is a short string (generated
  on your server when the image is uploaded) that decodes to a blurry preview of the image.
- `cachePolicy="memory-disk"` keeps decoded images in memory and files on disk, so scrolling back up
  a list or reopening the app doesn't refetch.
- In recycling lists (FlashList, Level 4) set `recyclingKey` to the item ID, so a recycled cell
  doesn't briefly show the previous item's picture.

### Don't download more pixels than you show

A 4000×3000 photo displayed at 120×90 points still has to be downloaded and decoded at full size —
that's tens of megabytes of memory for one thumbnail. If your image server can resize, request the
size you need:

```ts
import { PixelRatio } from 'react-native';

export function sizedUrl(base: string, widthPoints: number) {
  const px = PixelRatio.getPixelSizeForLayoutSize(widthPoints); // points → physical pixels
  return `${base}?w=${px}`; // the query format depends on your image CDN
}
```

TheMealDB, used in this course's examples, serves smaller variants when you append a size segment to
a thumbnail URL. Checked while writing this lesson, one recipe's full JPEG was about 134 KB, while
`/large`, `/medium` and `/small` returned roughly 71 KB, 40 KB and 9 KB. Check your own API's
documentation — the pattern differs between services.

## Custom fonts

```bash
npx expo install expo-font expo-splash-screen
```

The simplest, most robust way is to **embed fonts at build time** with the `expo-font` config
plugin, so they're available before any JavaScript runs:

```json title="app.json (excerpt)"
{
  "expo": {
    "plugins": [
      ["expo-font", { "fonts": ["./assets/fonts/Inter-Regular.ttf", "./assets/fonts/Inter-Bold.ttf"] }]
    ]
  }
}
```

Embedding requires a native build (a development build or EAS Build — Level 3). In Expo Go, or if
you prefer loading at runtime, use `useFonts` and keep the splash screen up until fonts are ready:

```tsx title="app/_layout.tsx"
import { useEffect } from 'react';
import { Stack } from 'expo-router';
import { useFonts } from 'expo-font';
import * as SplashScreen from 'expo-splash-screen';

SplashScreen.preventAutoHideAsync(); // call at module scope, before the first render

export default function RootLayout() {
  const [loaded, error] = useFonts({
    'Inter-Regular': require('../assets/fonts/Inter-Regular.ttf'),
    'Inter-Bold': require('../assets/fonts/Inter-Bold.ttf'),
  });

  useEffect(() => {
    if (loaded || error) SplashScreen.hideAsync();
  }, [loaded, error]);

  if (!loaded && !error) return null; // the native splash is still covering the screen

  return <Stack screenOptions={{ headerTitleStyle: { fontFamily: 'Inter-Bold' } }} />;
}
```

Use the font by its registered name: `fontFamily: 'Inter-Bold'`. Note that on Android, a custom
font with `fontWeight: '700'` doesn't magically pick the bold file — register each weight as its own
family name and choose it explicitly. If loading fails (`error`), the app still continues with the
system font instead of staying on the splash screen forever.

[Inter](https://rsms.me/inter/) is released under the SIL Open Font License, which allows bundling
it in an app; check the license of any font before you ship it.

## Icons

`@expo/vector-icons` bundles popular icon sets (Ionicons, MaterialIcons, Feather and more) as
fonts:

```tsx
import Ionicons from '@expo/vector-icons/Ionicons';

<Ionicons name="checkmark-circle" size={24} color="#16a34a" accessibilityLabel="Done" />
```

Icons rendered as font glyphs are sharp at every size and cheap to colour. Importing from the
per-set path (`@expo/vector-icons/Ionicons`) avoids pulling every set into your bundle.

## App icon and splash screen

These are configured in `app.json` and generated into the native projects at build time:

```json title="app.json (excerpt)"
{
  "expo": {
    "icon": "./assets/icon.png",
    "plugins": [
      ["expo-splash-screen", {
        "image": "./assets/splash-icon.png",
        "imageWidth": 200,
        "backgroundColor": "#ffffff",
        "dark": { "backgroundColor": "#020617" }
      }]
    ],
    "android": {
      "adaptiveIcon": { "foregroundImage": "./assets/android-icon-foreground.png", "backgroundColor": "#E6F4FE" }
    }
  }
}
```

Provide a large square master icon (1024×1024 is the common requirement for the App Store) and let
the build generate the sizes. Android's **adaptive icons** split the icon into a foreground and
background layer, so launchers can mask it into circles, squircles or teardrops — keep important
content inside the central safe zone of the foreground image.

## Worked example: a recipe card with a graceful image

```tsx title="src/RecipeCard.tsx"
import { memo } from 'react';
import { Pressable, StyleSheet, Text, View } from 'react-native';
import { Image } from 'expo-image';
import Ionicons from '@expo/vector-icons/Ionicons';

type Props = { id: string; name: string; thumbnail: string; area: string | null; favourite: boolean; onPress: () => void };

export const RecipeCard = memo(function RecipeCard({ id, name, thumbnail, area, favourite, onPress }: Props) {
  return (
    <Pressable onPress={onPress} style={styles.card} accessibilityRole="button" accessibilityLabel={`${name}${favourite ? ', favourite' : ''}`}>
      <Image
        source={{ uri: `${thumbnail}/medium` }}
        recyclingKey={id}
        style={styles.image}
        contentFit="cover"
        transition={150}
        cachePolicy="memory-disk"
      />
      <View style={styles.meta}>
        <Text style={styles.name} numberOfLines={2}>{name}</Text>
        {area && <Text style={styles.area}>{area}</Text>}
      </View>
      {favourite && <Ionicons name="heart" size={20} color="#e11d48" style={styles.heart} />}
    </Pressable>
  );
});

const styles = StyleSheet.create({
  card: { flex: 1, backgroundColor: 'white', borderRadius: 14, overflow: 'hidden' },
  image: { width: '100%', aspectRatio: 1, backgroundColor: '#e2e8f0' },
  meta: { padding: 10, gap: 2 },
  name: { fontSize: 15, fontWeight: '600' },
  area: { fontSize: 13, color: '#64748b' },
  heart: { position: 'absolute', top: 8, right: 8 },
});
```

The grey `backgroundColor` on the image keeps the grid stable while images load or if one fails.

## How It Actually Works

When Metro bundles `require('./empty-state.png')`, it doesn't inline the image. It registers an
**asset** record — file hashes, the available scales (1, 2, 3), width and height — and the `require`
call returns a numeric asset ID. In development, the app fetches the right scale from Metro's HTTP
server; in a release build, the files are copied into the app bundle (iOS) or `res/drawable-*dpi`
folders (Android), and the OS's resource system picks the scale for the device.

`expo-image` delegates to mature native image libraries — **SDWebImage** on iOS and **Glide** on
Android — which download on background threads, decode off the main thread, downsample to the view's
size, and maintain the memory and disk caches. That's why it handles big images and fast-scrolling
lists better than a naive implementation.

Fonts embedded through the config plugin are added to the native project (listed in iOS's
`Info.plist` and copied to Android's assets) so the OS registers them at launch. `useFonts` instead
loads the file at runtime and registers it with the platform's font manager, which takes a moment —
hence the splash screen. `preventAutoHideAsync` simply tells the native side not to remove the
launch screen when the first React frame renders; `hideAsync` removes it.

## Common mistakes

- **Dynamic `require` paths** — Metro can't resolve them.
- **Remote images without a size or aspect ratio** — invisible (Level 1, lesson 2).
- **Huge source images for thumbnails** — memory spikes and crashes on low-end Android.
- **Forgetting `recyclingKey`** in recycled lists — flashes of the wrong image.
- **Never hiding the splash screen** on font errors — the app looks frozen.
- **Expecting `fontWeight` to select a custom font's bold file** on Android.
- **Icons without accessibility labels** when they carry meaning.

## Exercise

1. Build a two-column recipe grid using `RecipeCard`, fed by the search query from lesson 5.
2. Add `@2x` and `@3x` variants of an empty-state illustration and show it in
   `ListEmptyComponent`. Confirm it's sharp on your device.
3. Add a custom font with `useFonts` and the splash-screen pattern. Simulate a failure by pointing
   one font at a remote URL that returns 404 (`{ 'Broken': 'https://example.com/missing.ttf' }`) —
   does your app still get past the splash screen?
4. Measure: open the performance monitor from the dev menu, fling the grid quickly with and without
   `cachePolicy="memory-disk"`, and write down what you observe on your device (no need to quote
   numbers you didn't see).
