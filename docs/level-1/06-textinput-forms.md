---
description: "Build forms with TextInput: controlled inputs, keyboard types, autofill hints, focusing the next field with refs, submit behaviour and inline validation."
---

# 06 · TextInput & Simple Forms

`TextInput` is the one core component that hands control to the operating system's keyboard, and the
keyboard has opinions. It can show a numeric pad or an email layout, autofill a saved password or
a one-time code, autocorrect, autocapitalize, and slide up over half your screen. Getting a form
right on mobile is mostly about telling the keyboard what you want.

## A controlled input

```tsx
import { useState } from 'react';
import { TextInput } from 'react-native';

function NameField() {
  const [name, setName] = useState('');
  return (
    <TextInput
      value={name}
      onChangeText={setName}
      placeholder="Habit name"
      style={{ borderWidth: 1, borderColor: '#cbd5e1', borderRadius: 10, padding: 12, fontSize: 16 }}
    />
  );
}
```

`onChangeText` gives you the string directly. (`onChange` gives a native event whose
`nativeEvent.text` holds the same thing — rarely needed.)

## Tell the keyboard what this field is

| Prop | Example values | Effect |
|------|----------------|--------|
| `keyboardType` | `default`, `email-address`, `number-pad`, `decimal-pad`, `phone-pad`, `url` | Which keyboard layout appears |
| `inputMode` | `text`, `numeric`, `decimal`, `email`, `tel`, `url`, `search` | Web-style alternative to `keyboardType`; takes precedence if both are set |
| `autoCapitalize` | `none`, `sentences`, `words`, `characters` | Shift behaviour |
| `autoCorrect` | `false` for emails, usernames, codes | Stops "fixing" user input |
| `autoComplete` | `email`, `password`, `new-password`, `one-time-code`, `name`, `postal-code` | Autofill hint (Android and iOS) |
| `textContentType` | `emailAddress`, `password`, `newPassword`, `oneTimeCode` | iOS-specific autofill hint |
| `secureTextEntry` | `true` | Masks input; enables password autofill |
| `returnKeyType` | `next`, `done`, `go`, `search`, `send` | Label on the return key |
| `maxLength` | `4` | Hard character limit |
| `multiline` | `true` | Textarea behaviour (return inserts a newline) |

An email field done properly:

```tsx
<TextInput
  value={email}
  onChangeText={setEmail}
  placeholder="you@example.com"
  keyboardType="email-address"
  autoCapitalize="none"
  autoCorrect={false}
  autoComplete="email"
  textContentType="emailAddress"
  returnKeyType="next"
/>
```

Without `autoCapitalize="none"`, iOS capitalises the first letter of every email address, and your
login fails for half your users.

## Moving between fields

A good mobile form lets the user press "next" on the keyboard to jump to the following field, and
"done" on the last field submits. You do that with refs:

```tsx
const passwordRef = useRef<TextInput>(null);

<TextInput returnKeyType="next" onSubmitEditing={() => passwordRef.current?.focus()} submitBehavior="submit" />
<TextInput ref={passwordRef} secureTextEntry returnKeyType="done" onSubmitEditing={handleSubmit} />
```

`submitBehavior="submit"` keeps the keyboard open while focus moves (instead of briefly closing and
reopening it). It replaced the older `blurOnSubmit` prop.

## Worked example: an "add habit" form with validation

```tsx title="AddHabitForm.tsx"
import { useRef, useState } from 'react';
import { Pressable, StyleSheet, Text, TextInput, View } from 'react-native';

export type NewHabit = { name: string; targetPerWeek: number };

type Errors = Partial<Record<keyof NewHabit, string>>;

export function validate(name: string, target: string): Errors {
  const errors: Errors = {};
  if (name.trim().length < 2) errors.name = 'Give the habit a name (2+ characters).';
  const n = Number(target);
  if (!Number.isInteger(n) || n < 1 || n > 7) errors.targetPerWeek = 'Enter a whole number from 1 to 7.';
  return errors;
}

export function AddHabitForm({ onSubmit }: { onSubmit: (h: NewHabit) => void }) {
  const [name, setName] = useState('');
  const [target, setTarget] = useState('7');
  const [touched, setTouched] = useState(false);
  const targetRef = useRef<TextInput>(null);

  const errors = validate(name, target);
  const valid = Object.keys(errors).length === 0;

  function submit() {
    setTouched(true);
    if (!valid) return;
    onSubmit({ name: name.trim(), targetPerWeek: Number(target) });
    setName('');
    setTarget('7');
    setTouched(false);
  }

  return (
    <View style={styles.form}>
      <Text style={styles.label}>Habit</Text>
      <TextInput
        value={name}
        onChangeText={setName}
        placeholder="e.g. Walk 8,000 steps"
        autoCapitalize="sentences"
        returnKeyType="next"
        submitBehavior="submit"
        onSubmitEditing={() => targetRef.current?.focus()}
        accessibilityLabel="Habit name"
        style={[styles.input, touched && errors.name && styles.inputError]}
      />
      {touched && errors.name && <Text style={styles.error}>{errors.name}</Text>}

      <Text style={styles.label}>Days per week</Text>
      <TextInput
        ref={targetRef}
        value={target}
        onChangeText={(t) => setTarget(t.replace(/[^0-9]/g, ''))}
        keyboardType="number-pad"
        maxLength={1}
        returnKeyType="done"
        onSubmitEditing={submit}
        style={[styles.input, touched && errors.targetPerWeek && styles.inputError]}
      />
      {touched && errors.targetPerWeek && <Text style={styles.error}>{errors.targetPerWeek}</Text>}

      <Pressable onPress={submit} style={({ pressed }) => [styles.button, pressed && { opacity: 0.8 }]} accessibilityRole="button">
        <Text style={styles.buttonText}>Add habit</Text>
      </Pressable>
    </View>
  );
}

const styles = StyleSheet.create({
  form: { gap: 8, padding: 16 },
  label: { fontSize: 13, fontWeight: '600', color: '#475569' },
  input: { borderWidth: 1, borderColor: '#cbd5e1', borderRadius: 10, paddingHorizontal: 12, paddingVertical: 10, fontSize: 16, backgroundColor: 'white' },
  inputError: { borderColor: '#dc2626' },
  error: { color: '#dc2626', fontSize: 13 },
  button: { marginTop: 12, backgroundColor: '#4f46e5', borderRadius: 12, paddingVertical: 14, alignItems: 'center' },
  buttonText: { color: 'white', fontWeight: '700', fontSize: 16 },
});
```

Design choices worth noticing:

- Errors only show after the first submit attempt (`touched`), so users aren't shouted at while
  typing their first letter.
- `number-pad` has no minus or decimal on most keyboards, but users can still paste text — so the
  input filters non-digits *and* `validate` checks the value. Never trust the keyboard type as
  validation.
- `validate` is a plain function, which makes it trivial to unit-test (Level 4).

## How It Actually Works

`TextInput` wraps a native text field (`UITextField`/`UITextView` on iOS, `EditText` on Android). The
native field owns the text while the user types; each keystroke goes to the native view first and
then an event is sent to JavaScript, where your `onChangeText` sets state, React re-renders, and the
new `value` prop goes back to native.

That round trip creates a race: if the user types faster than JavaScript responds, a stale `value`
could overwrite newer native text. React Native prevents this with an **event counter**: every
change event from native carries a count, and when JavaScript sends a `value` back it includes the
count it was responding to. Native ignores updates based on an older count. That's why a controlled
`TextInput` usually doesn't drop characters even under load — and also why transforming the text in
`onChangeText` (like the digit filter above) can cause a visible flicker when JS is slow: the
character appears natively, then disappears when JS catches up.

Focus is native too: `ref.current.focus()` calls into the platform view, which asks the OS to show
the keyboard. The keyboard itself is a separate system window that overlaps your app — it doesn't
resize your layout automatically. Handling that is lesson L2-09.

## Common mistakes

- **Email and username fields with autocapitalization and autocorrect on.**
- **Using `keyboardType="numeric"` as validation** — paste bypasses it; validate the value.
- **Forgetting `secureTextEntry` + `autoComplete="password"`**, which blocks password managers.
- **`multiline` inputs with `returnKeyType="done"` expecting submit** — in a multiline field return
  inserts a newline. Provide a button.
- **Showing errors on every keystroke** before the user has finished typing.
- **Placeholder as the only label** — it vanishes once the user types and is a poor accessible
  name. Use a visible label.

## Exercise

Build a sign-up form with four fields: name, email, password and a 6-digit invite code. Each field
must use the right keyboard, capitalisation and autofill hints (`new-password` for the password,
`one-time-code` for the code). Pressing "next" moves through all four fields; "done" on the last
submits. Validate on submit: email contains `@` and a dot after it, password is at least 8
characters, code is exactly 6 digits. Write `validate` as a pure function you could unit-test.
