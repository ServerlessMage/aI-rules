---
description: 
globs: 
alwaysApply: true
---
Choose the appropriate state management solution based on complexity and scope. Prefer local state when possible.

Use useState for simple local component state:

```tsx
// GOOD - local state for simple UI interactions
const Counter = () => {
  const [count, setCount] = useState<number>(0);
  
  return (
    <div>
      <span>{count}</span>
      <button onClick={() => setCount(count + 1)}>Increment</button>
    </div>
  );
};
```

Lift state up to the nearest common ancestor when multiple components need access:

```tsx
// GOOD - lifted state
const TodoApp = () => {
  const [todos, setTodos] = useState<Todo[]>([]);
  
  const addTodo = (text: string) => {
    const newTodo: Todo = {
      id: Date.now().toString(),
      text,
      completed: false,
    };
    setTodos([...todos, newTodo]);
  };
  
  const toggleTodo = (id: string) => {
    setTodos(todos.map(todo =>
      todo.id === id ? { ...todo, completed: !todo.completed } : todo
    ));
  };
  
  return (
    <div>
      <TodoForm onAdd={addTodo} />
      <TodoList todos={todos} onToggle={toggleTodo} />
    </div>
  );
};
```

Use Context API for moderate complexity shared state:

```tsx
// GOOD - Context for shared state
interface AuthState {
  readonly user: User | null;
  readonly loading: boolean;
}

interface AuthContextType extends AuthState {
  readonly login: (email: string, password: string) => Promise<void>;
  readonly logout: () => void;
}

const AuthContext = createContext<AuthContextType | null>(null);

const AuthProvider = ({ children }: { children: React.ReactNode }) => {
  const [state, setState] = useState<AuthState>({
    user: null,
    loading: true,
  });
  
  const login = async (email: string, password: string) => {
    setState(prev => ({ ...prev, loading: true }));
    try {
      const user = await authService.login(email, password);
      setState({ user, loading: false });
    } catch (error) {
      setState({ user: null, loading: false });
      throw error;
    }
  };
  
  const logout = () => {
    authService.logout();
    setState({ user: null, loading: false });
  };
  
  return (
    <AuthContext.Provider value={{ ...state, login, logout }}>
      {children}
    </AuthContext.Provider>
  );
};

// Custom hook for using auth context
const useAuth = () => {
  const context = useContext(AuthContext);
  if (!context) {
    throw new Error('useAuth must be used within an AuthProvider');
  }
  return context;
};
```

Avoid prop drilling by using composition:

```tsx
// BAD - prop drilling
const App = () => {
  const theme = useTheme();
  return <Layout theme={theme} />;
};

const Layout = ({ theme }) => <Sidebar theme={theme} />;
const Sidebar = ({ theme }) => <ThemeToggle theme={theme} />;
```

```tsx
// GOOD - composition with context
const ThemeContext = createContext<Theme>('light');

const App = () => {
  const theme = useTheme();
  return (
    <ThemeContext.Provider value={theme}>
      <Layout />
    </ThemeContext.Provider>
  );
};

const Layout = () => <Sidebar />;
const Sidebar = () => <ThemeToggle />;
const ThemeToggle = () => {
  const theme = useContext(ThemeContext);
  return <button>Current theme: {theme}</button>;
};
```

Use external state management for complex global state:

```tsx
// GOOD - Zustand for global state
interface AppState {
  readonly user: User | null;
  readonly todos: Todo[];
  readonly login: (user: User) => void;
  readonly logout: () => void;
  readonly addTodo: (text: string) => void;
  readonly toggleTodo: (id: string) => void;
}

const useAppStore = create<AppState>((set) => ({
  user: null,
  todos: [],
  
  login: (user) => set({ user }),
  logout: () => set({ user: null }),
  
  addTodo: (text) => set((state) => ({
    todos: [...state.todos, {
      id: Date.now().toString(),
      text,
      completed: false,
    }],
  })),
  
  toggleTodo: (id) => set((state) => ({
    todos: state.todos.map(todo =>
      todo.id === id ? { ...todo, completed: !todo.completed } : todo
    ),
  })),
}));

// Usage in components
const TodoList = () => {
  const { todos, toggleTodo } = useAppStore();
  
  return (
    <ul>
      {todos.map(todo => (
        <li key={todo.id}>
          <input
            type="checkbox"
            checked={todo.completed}
            onChange={() => toggleTodo(todo.id)}
          />
          {todo.text}
        </li>
      ))}
    </ul>
  );
};
```

Use state machines for complex state transitions:

```tsx
// GOOD - state machine pattern
type FetchState = 
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'success'; data: User[] }
  | { status: 'error'; error: string };

const useFetchUsers = () => {
  const [state, setState] = useState<FetchState>({ status: 'idle' });
  
  const fetchUsers = async () => {
    setState({ status: 'loading' });
    try {
      const data = await api.getUsers();
      setState({ status: 'success', data });
    } catch (error) {
      setState({ status: 'error', error: error.message });
    }
  };
  
  const reset = () => setState({ status: 'idle' });
  
  return { state, fetchUsers, reset };
};
```

Guidelines for choosing state management:
- **Local state (useState)**: Component-specific UI state, form inputs
- **Lifted state**: Shared between 2-3 closely related components
- **Context**: App-wide state like authentication, theme, language
- **External libraries**: Complex state with many interdependencies