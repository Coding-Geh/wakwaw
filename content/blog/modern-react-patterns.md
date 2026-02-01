---
title: "Modern React Patterns yang Wajib Kamu Tahu"
date: 2025-09-01
draft: false
tags: ["React", "JavaScript", "Frontend", "Design Patterns"]
categories: ["Frontend"]
author: "wakwaw"
showToc: true
TocOpen: false
description: "Pelajari patterns modern di React yang akan meningkatkan kualitas code kamu"
cover:
  image: "https://images.unsplash.com/photo-1633356122544-f134324a6cee?w=1200&h=630&fit=crop"
  alt: "React Patterns"
  caption: "Modern React development patterns"
  relative: false
---

## Introduction

React terus berkembang dan patterns yang kita gunakan juga harus ikut berkembang. Artikel ini membahas modern patterns yang akan membuat code React kamu lebih clean dan maintainable.

## 1. Compound Components Pattern

Pattern ini memungkinkan components yang saling terkait untuk berbagi state secara implisit:

```tsx
// Usage
<Select onChange={handleChange}>
  <Select.Trigger>
    <Select.Value placeholder="Select option" />
  </Select.Trigger>
  <Select.Content>
    <Select.Item value="react">React</Select.Item>
    <Select.Item value="vue">Vue</Select.Item>
    <Select.Item value="svelte">Svelte</Select.Item>
  </Select.Content>
</Select>
```

Implementation:

```tsx
import { createContext, useContext, useState, ReactNode } from 'react';

// Context for shared state
interface SelectContextType {
  value: string;
  onChange: (value: string) => void;
  isOpen: boolean;
  setIsOpen: (open: boolean) => void;
}

const SelectContext = createContext<SelectContextType | null>(null);

// Hook to access context
function useSelectContext() {
  const context = useContext(SelectContext);
  if (!context) {
    throw new Error('Select components must be used within Select');
  }
  return context;
}

// Main component
interface SelectProps {
  children: ReactNode;
  onChange: (value: string) => void;
  defaultValue?: string;
}

function Select({ children, onChange, defaultValue = '' }: SelectProps) {
  const [value, setValue] = useState(defaultValue);
  const [isOpen, setIsOpen] = useState(false);

  const handleChange = (newValue: string) => {
    setValue(newValue);
    onChange(newValue);
    setIsOpen(false);
  };

  return (
    <SelectContext.Provider value={{ value, onChange: handleChange, isOpen, setIsOpen }}>
      <div className="relative">{children}</div>
    </SelectContext.Provider>
  );
}

// Sub-components
Select.Trigger = function Trigger({ children }: { children: ReactNode }) {
  const { setIsOpen, isOpen } = useSelectContext();
  return (
    <button onClick={() => setIsOpen(!isOpen)} className="select-trigger">
      {children}
    </button>
  );
};

Select.Value = function Value({ placeholder }: { placeholder: string }) {
  const { value } = useSelectContext();
  return <span>{value || placeholder}</span>;
};

Select.Content = function Content({ children }: { children: ReactNode }) {
  const { isOpen } = useSelectContext();
  if (!isOpen) return null;
  return <div className="select-content">{children}</div>;
};

Select.Item = function Item({ value, children }: { value: string; children: ReactNode }) {
  const { onChange, value: selectedValue } = useSelectContext();
  return (
    <div
      onClick={() => onChange(value)}
      className={`select-item ${selectedValue === value ? 'selected' : ''}`}
    >
      {children}
    </div>
  );
};

export { Select };
```

## 2. Render Props Pattern (Modern Version)

Meskipun hooks menggantikan banyak use cases, render props masih berguna:

```tsx
interface MousePosition {
  x: number;
  y: number;
}

interface MouseTrackerProps {
  children: (position: MousePosition) => ReactNode;
}

function MouseTracker({ children }: MouseTrackerProps) {
  const [position, setPosition] = useState<MousePosition>({ x: 0, y: 0 });

  useEffect(() => {
    const handleMove = (e: MouseEvent) => {
      setPosition({ x: e.clientX, y: e.clientY });
    };

    window.addEventListener('mousemove', handleMove);
    return () => window.removeEventListener('mousemove', handleMove);
  }, []);

  return <>{children(position)}</>;
}

// Usage
<MouseTracker>
  {({ x, y }) => (
    <div>Mouse position: {x}, {y}</div>
  )}
</MouseTracker>
```

## 3. Custom Hooks Pattern

Encapsulate reusable logic dalam custom hooks:

```tsx
// useLocalStorage hook
function useLocalStorage<T>(key: string, initialValue: T) {
  const [storedValue, setStoredValue] = useState<T>(() => {
    if (typeof window === 'undefined') return initialValue;
    
    try {
      const item = window.localStorage.getItem(key);
      return item ? JSON.parse(item) : initialValue;
    } catch (error) {
      console.error(error);
      return initialValue;
    }
  });

  const setValue = (value: T | ((val: T) => T)) => {
    try {
      const valueToStore = value instanceof Function ? value(storedValue) : value;
      setStoredValue(valueToStore);
      
      if (typeof window !== 'undefined') {
        window.localStorage.setItem(key, JSON.stringify(valueToStore));
      }
    } catch (error) {
      console.error(error);
    }
  };

  return [storedValue, setValue] as const;
}

// Usage
function App() {
  const [theme, setTheme] = useLocalStorage('theme', 'light');
  
  return (
    <button onClick={() => setTheme(theme === 'light' ? 'dark' : 'light')}>
      Current: {theme}
    </button>
  );
}
```

## 4. Component Composition with Slots

Pattern ini terinspirasi dari Vue's slots:

```tsx
interface CardProps {
  children: ReactNode;
}

interface SlotProps {
  children: ReactNode;
}

function Card({ children }: CardProps) {
  const slots = {
    header: null as ReactNode,
    body: null as ReactNode,
    footer: null as ReactNode,
  };

  // Extract slots from children
  React.Children.forEach(children, (child) => {
    if (React.isValidElement(child)) {
      if (child.type === Card.Header) slots.header = child;
      if (child.type === Card.Body) slots.body = child;
      if (child.type === Card.Footer) slots.footer = child;
    }
  });

  return (
    <div className="card">
      {slots.header && <div className="card-header">{slots.header}</div>}
      {slots.body && <div className="card-body">{slots.body}</div>}
      {slots.footer && <div className="card-footer">{slots.footer}</div>}
    </div>
  );
}

Card.Header = ({ children }: SlotProps) => <>{children}</>;
Card.Body = ({ children }: SlotProps) => <>{children}</>;
Card.Footer = ({ children }: SlotProps) => <>{children}</>;

// Usage
<Card>
  <Card.Header>
    <h2>Card Title</h2>
  </Card.Header>
  <Card.Body>
    <p>Card content goes here</p>
  </Card.Body>
  <Card.Footer>
    <button>Action</button>
  </Card.Footer>
</Card>
```

## 5. State Reducer Pattern

Memberikan control kepada user untuk customize state updates:

```tsx
type ToggleState = { on: boolean };
type ToggleAction = { type: 'toggle' } | { type: 'reset' };

interface UseToggleOptions {
  reducer?: (state: ToggleState, action: ToggleAction) => ToggleState;
  initialOn?: boolean;
}

function defaultReducer(state: ToggleState, action: ToggleAction): ToggleState {
  switch (action.type) {
    case 'toggle':
      return { on: !state.on };
    case 'reset':
      return { on: false };
    default:
      return state;
  }
}

function useToggle({ reducer = defaultReducer, initialOn = false }: UseToggleOptions = {}) {
  const [{ on }, dispatch] = useReducer(reducer, { on: initialOn });

  const toggle = () => dispatch({ type: 'toggle' });
  const reset = () => dispatch({ type: 'reset' });

  return { on, toggle, reset };
}

// Usage with custom reducer
function App() {
  const { on, toggle, reset } = useToggle({
    reducer: (state, action) => {
      // Custom logic: can only toggle once
      if (action.type === 'toggle' && state.on) {
        return state; // Prevent turning off
      }
      return defaultReducer(state, action);
    },
  });

  return (
    <div>
      <button onClick={toggle}>{on ? 'ON' : 'OFF'}</button>
      <button onClick={reset}>Reset</button>
    </div>
  );
}
```

## 6. Polymorphic Components Pattern

Komponen yang bisa render sebagai element berbeda:

```tsx
type PolymorphicRef<C extends React.ElementType> = React.ComponentPropsWithRef<C>['ref'];

type AsProp<C extends React.ElementType> = {
  as?: C;
};

type PropsToOmit<C extends React.ElementType, P> = keyof (AsProp<C> & P);

type PolymorphicComponentProp<
  C extends React.ElementType,
  Props = {}
> = React.PropsWithChildren<Props & AsProp<C>> &
  Omit<React.ComponentPropsWithoutRef<C>, PropsToOmit<C, Props>>;

// Button component
interface ButtonProps {
  variant?: 'primary' | 'secondary';
}

type ButtonComponentProps<C extends React.ElementType> = PolymorphicComponentProp<C, ButtonProps>;

function Button<C extends React.ElementType = 'button'>({
  as,
  children,
  variant = 'primary',
  ...props
}: ButtonComponentProps<C>) {
  const Component = as || 'button';
  
  return (
    <Component className={`btn btn-${variant}`} {...props}>
      {children}
    </Component>
  );
}

// Usage
<Button>Click me</Button> // renders as <button>
<Button as="a" href="/home">Go Home</Button> // renders as <a>
<Button as={Link} to="/about">About</Button> // renders as React Router Link
```

## Conclusion

Patterns ini akan membantu kamu menulis code React yang lebih:
- **Reusable:** Components yang bisa dipakai ulang
- **Flexible:** Mudah dikustomisasi sesuai kebutuhan
- **Maintainable:** Mudah dipahami dan di-maintain

Start small, implement satu pattern di project kamu dan lihat perbedaannya! 🚀

---

*Ada pattern favorit kamu yang belum dibahas? Let me know di [@wakwaw](https://twitter.com/wakwaw)!*
