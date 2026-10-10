---
description: "Build larger forms with React Hook Form's Controller and a Zod schema: typed values, field errors, cross-field rules, focus on error and tested validation."
---

# 07 · Forms & Validation with React Hook Form and Zod

Level 1's hand-rolled form — `useState` per field, a `validate` function, a `touched` flag — is fine
for three fields. At ten fields with cross-field rules ("end date after start date"), async checks,
and the need to focus the first invalid field, it turns into a lot of bookkeeping. **React Hook
Form** handles form state and **Zod** describes the rules as a schema that also produces your
TypeScript types.

```bash
npx expo install react-hook-form zod @hookform/resolvers
```

## The schema is the source of truth

```ts title="src/habitSchema.ts"
import { z } from 'zod';

export const habitSchema = z
  .object({
    name: z.string().trim().min(2, 'Use at least 2 characters.').max(40, 'Keep it under 40 characters.'),
    frequency: z.enum(['daily', 'weekly']),
    timesPerWeek: z.coerce.number().int('Whole numbers only.').min(1).max(7),
    reminder: z.boolean(),
    reminderTime: z
      .string()
      .regex(/^([01]\d|2[0-3]):[0-5]\d$/, 'Use 24-hour HH:MM.')
      .optional()
      .or(z.literal('')),
    startDate: z.string().regex(/^\d{4}-\d{2}-\d{2}$/, 'Use YYYY-MM-DD.'),
    endDate: z.string().regex(/^\d{4}-\d{2}-\d{2}$/, 'Use YYYY-MM-DD.').optional().or(z.literal('')),
  })
  .refine((v) => !v.reminder || !!v.reminderTime, {
    path: ['reminderTime'],
    message: 'Pick a reminder time, or turn reminders off.',
  })
  .refine((v) => !v.endDate || v.endDate >= v.startDate, {
    path: ['endDate'],
    message: 'End date must be on or after the start date.',
  });

export type HabitFormInput = z.input<typeof habitSchema>;   // what the fields hold (strings from TextInput)
export type HabitFormValues = z.output<typeof habitSchema>; // what you get after parsing (numbers, trimmed)
```

Things to notice:

- `z.coerce.number()` turns the string a `TextInput` produces into a number, then validates it.
- `.trim()` normalises before checking length, so `"  a "` fails.
- `.refine` adds **cross-field rules**; `path` attaches the error to the field the user should fix.
- Comparing `YYYY-MM-DD` strings with `>=` works because that format sorts lexicographically in date
  order.
- `z.input` and `z.output` give you two types for free: the raw form values and the parsed result.

Because the schema is plain TypeScript, you can test it without React. These tests were run with
Jest (zod 4.6, react-hook-form 7.89) and passed:

```ts title="src/habitSchema.test.ts"
import { habitSchema } from './habitSchema';

const base = {
  name: 'Read', frequency: 'weekly', timesPerWeek: '3', reminder: false,
  reminderTime: '', startDate: '2026-10-01', endDate: '',
};

test('valid input parses and coerces', () => {
  const r = habitSchema.safeParse(base);
  expect(r.success).toBe(true);
  if (r.success) expect(r.data.timesPerWeek).toBe(3);
});

test('reminder on without a time is rejected on the right field', () => {
  const r = habitSchema.safeParse({ ...base, reminder: true });
  expect(r.success).toBe(false);
  if (!r.success) expect(r.error.issues[0].path).toEqual(['reminderTime']);
});

test('end date before start date is rejected', () => {
  const r = habitSchema.safeParse({ ...base, endDate: '2026-09-30' });
  expect(r.success).toBe(false);
  if (!r.success) expect(r.error.issues.map((i) => i.path[0])).toContain('endDate');
});

test('name is trimmed before length check', () => {
  expect(habitSchema.safeParse({ ...base, name: '  a ' }).success).toBe(false);
});
```

## Connecting inputs: Controller

React Native's `TextInput` isn't a DOM input, so React Hook Form's `register` (which attaches to DOM
elements) doesn't apply. Use `Controller`, or a small reusable wrapper around it:

```tsx title="src/FormField.tsx"
import { Control, Controller, FieldPath, FieldValues } from 'react-hook-form';
import { StyleSheet, Text, TextInput, TextInputProps, View } from 'react-native';

type Props<T extends FieldValues> = {
  control: Control<T>;
  name: FieldPath<T>;
  label: string;
} & Omit<TextInputProps, 'value' | 'onChangeText' | 'onBlur'>;

export function FormField<T extends FieldValues>({ control, name, label, ...inputProps }: Props<T>) {
  return (
    <Controller
      control={control}
      name={name}
      render={({ field: { value, onChange, onBlur, ref }, fieldState: { error } }) => (
        <View style={styles.field}>
          <Text style={styles.label}>{label}</Text>
          <TextInput
            ref={ref}
            value={value == null ? '' : String(value)}
            onChangeText={onChange}
            onBlur={onBlur}
            accessibilityLabel={label}
            style={[styles.input, error && styles.inputError]}
            {...inputProps}
          />
          {error?.message ? <Text style={styles.error} accessibilityLiveRegion="polite">{error.message}</Text> : null}
        </View>
      )}
    />
  );
}

const styles = StyleSheet.create({
  field: { gap: 4 },
  label: { fontSize: 13, fontWeight: '600', color: '#475569' },
  input: { borderWidth: 1, borderColor: '#cbd5e1', borderRadius: 10, paddingHorizontal: 12, paddingVertical: 10, fontSize: 16, backgroundColor: 'white' },
  inputError: { borderColor: '#dc2626' },
  error: { color: '#dc2626', fontSize: 13 },
});
```

Passing `field.ref` to the `TextInput` lets React Hook Form **focus the first invalid field** when
submit fails (`shouldFocusError` is on by default).

## Worked example: the habit form

```tsx title="app/add.tsx"
import { useForm, Controller } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { Pressable, ScrollView, StyleSheet, Switch, Text, View } from 'react-native';
import { router } from 'expo-router';
import { habitSchema, HabitFormInput, HabitFormValues } from '../src/habitSchema';
import { FormField } from '../src/FormField';

export default function AddHabit() {
  const { control, handleSubmit, watch, formState: { isSubmitting } } = useForm<HabitFormInput, unknown, HabitFormValues>({
    resolver: zodResolver(habitSchema),
    defaultValues: {
      name: '', frequency: 'daily', timesPerWeek: '7', reminder: false,
      reminderTime: '', startDate: new Date().toISOString().slice(0, 10), endDate: '',
    },
    mode: 'onTouched', // validate a field after the user leaves it, then on every change
  });

  const reminder = watch('reminder');

  const onSubmit = async (values: HabitFormValues) => {
    // values.timesPerWeek is a number here, and values.name is trimmed
    console.log('saving', values);
    router.back();
  };

  return (
    <ScrollView contentContainerStyle={styles.form} keyboardShouldPersistTaps="handled">
      <FormField control={control} name="name" label="Habit name" placeholder="Walk 8,000 steps" returnKeyType="next" />
      <FormField control={control} name="timesPerWeek" label="Times per week" keyboardType="number-pad" maxLength={1} />
      <Controller
        control={control}
        name="reminder"
        render={({ field: { value, onChange } }) => (
          <View style={styles.switchRow}>
            <Text style={styles.switchLabel}>Daily reminder</Text>
            <Switch value={value} onValueChange={onChange} accessibilityLabel="Daily reminder" />
          </View>
        )}
      />
      {reminder && <FormField control={control} name="reminderTime" label="Reminder time (HH:MM)" placeholder="20:00" keyboardType="numbers-and-punctuation" />}
      <FormField control={control} name="startDate" label="Start date" placeholder="YYYY-MM-DD" />
      <FormField control={control} name="endDate" label="End date (optional)" placeholder="YYYY-MM-DD" />
      <Pressable
        onPress={handleSubmit(onSubmit)}
        disabled={isSubmitting}
        accessibilityRole="button"
        accessibilityState={{ disabled: isSubmitting }}
        style={({ pressed }) => [styles.button, (pressed || isSubmitting) && { opacity: 0.7 }]}
      >
        <Text style={styles.buttonText}>{isSubmitting ? 'Saving…' : 'Save habit'}</Text>
      </Pressable>
    </ScrollView>
  );
}

const styles = StyleSheet.create({
  form: { padding: 16, gap: 14 },
  switchRow: { flexDirection: 'row', alignItems: 'center', justifyContent: 'space-between', minHeight: 44 },
  switchLabel: { fontSize: 16 },
  button: { marginTop: 8, backgroundColor: '#4f46e5', borderRadius: 12, paddingVertical: 14, alignItems: 'center' },
  buttonText: { color: 'white', fontWeight: '700', fontSize: 16 },
});
```

Typing dates as text is a stand-in to keep this lesson about validation. In a real app you'd use a
native date picker (for example `@react-native-community/datetimepicker`) inside a `Controller` —
the schema doesn't change, because the picker would still produce a `YYYY-MM-DD` string.

## How It Actually Works

React Hook Form keeps form values in a **mutable store outside React state** (inside the object
`useForm` returns). A `Controller` subscribes only to its own field, so typing in "name" re-renders
the name field's `Controller` — not the whole form. `watch('reminder')` subscribes the parent to that
one field, which is why the form re-renders when the switch flips (and not when you type a name).

On submit, `handleSubmit` collects the current values and hands them to the **resolver**.
`zodResolver` parses the values with the schema (asynchronously by default, so async refinements
work); on failure it catches the Zod error and converts each Zod issue's
`path` and `message` into React Hook Form's error map (`errors.reminderTime.message`), on success it
passes Zod's **parsed output** — coerced and trimmed — to your `onSubmit`. That's why `onSubmit`
receives numbers even though every `TextInput` produced strings. With `mode: 'onTouched'`, the
resolver also runs when a field blurs for the first time and on each change after that, re-using the
same schema for live validation.

## Common mistakes

- **Using `register` with `TextInput`** — it's for DOM inputs. Use `Controller`.
- **Not passing `field.ref`** — focus-on-error silently doesn't happen.
- **Validating numbers without coercion** — `z.number()` rejects the string `"3"` from a
  `TextInput`.
- **Duplicating rules** in the UI and the schema — the schema should be the only place rules live,
  and it can be shared with your backend if it's also TypeScript.
- **Validating on every keystroke from the first character** (`mode: 'onChange'`) — noisy; prefer
  `onTouched` or `onBlur` for most mobile forms.
- **Forgetting `keyboardShouldPersistTaps="handled"`** on the scroll view — the first tap on the
  submit button only dismisses the keyboard.

## Exercise

1. Add a `category` field (`'health' | 'learning' | 'mindfulness' | 'other'`) rendered as a row of
   selectable chips inside a `Controller`.
2. Add a rule: if `frequency` is `'daily'`, `timesPerWeek` must be 7. Attach the error to
   `timesPerWeek` and write a test for it.
3. Add an **async** check that the name isn't already taken (simulate with a `Promise` that resolves
   after 300 ms against a local list) using Zod's `.refine(async …)` — then call `safeParseAsync` in
   your test.
4. Make the form an **edit** form too: when opened as `/add?id=42`, load the habit and pass it as
   `defaultValues` (hint: `reset(values)` once the data arrives).
