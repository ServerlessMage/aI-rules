---
description: 
globs: 
alwaysApply: true
---
Follow the Rules of Hooks and use them effectively for state management and side effects.

Always call hooks at the top level, never inside loops, conditions, or nested functions:

```tsx
// BAD - conditional hook usage
const MyComponent = ({ shouldFetch }: { shouldFetch: boolean }) => {
  if (shouldFetch) {
    const [data, setData] = useState(null); // Error!
  }
  
  return <div>Content</div>;
};
```

```tsx
// GOOD - hooks at top level
const MyComponent = ({ shouldFetch }: { shouldFetch: boolean }) => {
  const [data, setData] = useState(null);
  
  useEffect(() => {
    if (shouldFetch) {
      fetchData().then(setData);
    }
  }, [shouldFetch]);
  
  return <div>Content</div>;
};
```

Use useState with proper initial values and type annotations:

```tsx
// BAD - unclear state types
const [user, setUser] = useState();
const [count, setCount] = useState();
```

```tsx
// GOOD - explicit types and initial values
const [user, setUser] = useState<User | null>(null);
const [count, setCount] = useState<number>(0);
const [loading, setLoading] = useState<boolean>(false);
```

Use useEffect with proper dependency arrays:

```tsx
// BAD - missing dependencies
const UserProfile = ({ userId }: { userId: string }) => {
  const [user, setUser] = useState<User | null>(null);
  
  useEffect(() => {
    fetchUser(userId).then(setUser);
  }, []); // Missing userId dependency
  
  return <div>{user?.name}</div>;
};
```

```tsx
// GOOD - complete dependency array
const UserProfile = ({ userId }: { userId: string }) => {
  const [user, setUser] = useState<User | null>(null);
  
  useEffect(() => {
    fetchUser(userId).then(setUser);
  }, [userId]); // Includes all dependencies
  
  return <div>{user?.name}</div>;
};
```

Use useCallback and useMemo for performance optimization:

```tsx
// BAD - recreating functions on every render
const TodoList = ({ todos }: { todos: Todo[] }) => {
  const handleToggle = (id: string) => {
    // Toggle logic
  };
  
  const expensiveValue = todos.filter(todo => todo.completed).length;
  
  return (
    <div>
      {todos.map(todo => (
        <TodoItem key={todo.id} todo={todo} onToggle={handleToggle} />
      ))}
      <div>Completed: {expensiveValue}</div>
    </div>
  );
};
```

```tsx
// GOOD - memoized functions and values
const TodoList = ({ todos }: { todos: Todo[] }) => {
  const handleToggle = useCallback((id: string) => {
    // Toggle logic
  }, []);
  
  const completedCount = useMemo(
    () => todos.filter(todo => todo.completed).length,
    [todos]
  );
  
  return (
    <div>
      {todos.map(todo => (
        <TodoItem key={todo.id} todo={todo} onToggle={handleToggle} />
      ))}
      <div>Completed: {completedCount}</div>
    </div>
  );
};
```

Use useReducer for complex state logic:

```tsx
// GOOD - useReducer for complex state
interface State {
  readonly loading: boolean;
  readonly data: User[] | null;
  readonly error: string | null;
}

type Action =
  | { type: 'FETCH_START' }
  | { type: 'FETCH_SUCCESS'; payload: User[] }
  | { type: 'FETCH_ERROR'; payload: string };

const reducer = (state: State, action: Action): State => {
  switch (action.type) {
    case 'FETCH_START':
      return { ...state, loading: true, error: null };
    case 'FETCH_SUCCESS':
      return { loading: false, data: action.payload, error: null };
    case 'FETCH_ERROR':
      return { loading: false, data: null, error: action.payload };
    default:
      return state;
  }
};

const UserList = () => {
  const [state, dispatch] = useReducer(reducer, {
    loading: false,
    data: null,
    error: null,
  });
  
  useEffect(() => {
    dispatch({ type: 'FETCH_START' });
    fetchUsers()
      .then(users => dispatch({ type: 'FETCH_SUCCESS', payload: users }))
      .catch(error => dispatch({ type: 'FETCH_ERROR', payload: error.message }));
  }, []);
  
  if (state.loading) return <div>Loading...</div>;
  if (state.error) return <div>Error: {state.error}</div>;
  
  return (
    <ul>
      {state.data?.map(user => (
        <li key={user.id}>{user.name}</li>
      ))}
    </ul>
  );
};
```

Create custom hooks for reusable logic:

```tsx
// GOOD - custom hook
const useLocalStorage = <T>(key: string, initialValue: T) => {
  const [storedValue, setStoredValue] = useState<T>(() => {
    try {
      const item = window.localStorage.getItem(key);
      return item ? JSON.parse(item) : initialValue;
    } catch (error) {
      console.error(`Error reading localStorage key "${key}":`, error);
      return initialValue;
    }
  });
  
  const setValue = useCallback((value: T | ((val: T) => T)) => {
    try {
      const valueToStore = value instanceof Function ? value(storedValue) : value;
      setStoredValue(valueToStore);
      window.localStorage.setItem(key, JSON.stringify(valueToStore));
    } catch (error) {
      console.error(`Error setting localStorage key "${key}":`, error);
    }
  }, [key, storedValue]);
  
  return [storedValue, setValue] as const;
};

// Usage
const Settings = () => {
  const [theme, setTheme] = useLocalStorage('theme', 'light');
  
  return (
    <button onClick={() => setTheme(theme === 'light' ? 'dark' : 'light')}>
      Current theme: {theme}
    </button>
  );
};
```