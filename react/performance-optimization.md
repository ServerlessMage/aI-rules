---
description: 
globs: 
alwaysApply: true
---
Optimize React applications for better performance using memoization, code splitting, and proper rendering patterns.

Use React.memo to prevent unnecessary re-renders:

```tsx
// BAD - component re-renders even when props haven't changed
const ExpensiveComponent = ({ data, onUpdate }) => {
  const processedData = expensiveCalculation(data);
  return <div>{processedData}</div>;
};
```

```tsx
// GOOD - memoized component
interface ExpensiveComponentProps {
  readonly data: ComplexData;
  readonly onUpdate: (id: string) => void;
}

const ExpensiveComponent = React.memo(({ data, onUpdate }: ExpensiveComponentProps) => {
  const processedData = useMemo(() => expensiveCalculation(data), [data]);
  return <div>{processedData}</div>;
});

// Custom comparison function for complex props
const ExpensiveComponentWithCustomComparison = React.memo(
  ({ data, onUpdate }: ExpensiveComponentProps) => {
    const processedData = useMemo(() => expensiveCalculation(data), [data]);
    return <div>{processedData}</div>;
  },
  (prevProps, nextProps) => {
    return prevProps.data.id === nextProps.data.id && 
           prevProps.data.version === nextProps.data.version;
  }
);
```

Use useMemo for expensive calculations:

```tsx
// BAD - expensive calculation on every render
const DataVisualization = ({ rawData, filters }) => {
  const processedData = processLargeDataset(rawData, filters);
  const chartData = generateChartData(processedData);
  
  return <Chart data={chartData} />;
};
```

```tsx
// GOOD - memoized expensive calculations
const DataVisualization = ({ rawData, filters }: {
  rawData: RawData[];
  filters: FilterOptions;
}) => {
  const processedData = useMemo(
    () => processLargeDataset(rawData, filters),
    [rawData, filters]
  );
  
  const chartData = useMemo(
    () => generateChartData(processedData),
    [processedData]
  );
  
  return <Chart data={chartData} />;
};
```

Use useCallback to memoize functions:

```tsx
// BAD - new function on every render causes child re-renders
const TodoApp = () => {
  const [todos, setTodos] = useState<Todo[]>([]);
  
  const handleToggle = (id: string) => {
    setTodos(todos.map(todo =>
      todo.id === id ? { ...todo, completed: !todo.completed } : todo
    ));
  };
  
  return (
    <div>
      {todos.map(todo => (
        <TodoItem key={todo.id} todo={todo} onToggle={handleToggle} />
      ))}
    </div>
  );
};
```

```tsx
// GOOD - memoized callback
const TodoApp = () => {
  const [todos, setTodos] = useState<Todo[]>([]);
  
  const handleToggle = useCallback((id: string) => {
    setTodos(prevTodos => prevTodos.map(todo =>
      todo.id === id ? { ...todo, completed: !todo.completed } : todo
    ));
  }, []); // Empty dependency array since we use functional update
  
  return (
    <div>
      {todos.map(todo => (
        <TodoItem key={todo.id} todo={todo} onToggle={handleToggle} />
      ))}
    </div>
  );
};
```

Implement code splitting with React.lazy and Suspense:

```tsx
// GOOD - code splitting for route-based components
const HomePage = React.lazy(() => import('./pages/HomePage'));
const AboutPage = React.lazy(() => import('./pages/AboutPage'));
const ContactPage = React.lazy(() => import('./pages/ContactPage'));

const App = () => {
  return (
    <Router>
      <Suspense fallback={<div>Loading...</div>}>
        <Routes>
          <Route path="/" element={<HomePage />} />
          <Route path="/about" element={<AboutPage />} />
          <Route path="/contact" element={<ContactPage />} />
        </Routes>
      </Suspense>
    </Router>
  );
};
```

Use virtualization for large lists:

```tsx
// GOOD - virtualized list for performance
import { FixedSizeList as List } from 'react-window';

interface VirtualizedListProps {
  readonly items: ListItem[];
}

const VirtualizedList = ({ items }: VirtualizedListProps) => {
  const Row = ({ index, style }: { index: number; style: React.CSSProperties }) => (
    <div style={style}>
      <ListItem data={items[index]} />
    </div>
  );
  
  return (
    <List
      height={600}
      itemCount={items.length}
      itemSize={50}
      width="100%"
    >
      {Row}
    </List>
  );
};
```

Optimize context to prevent unnecessary re-renders:

```tsx
// BAD - single context causes all consumers to re-render
const AppContext = createContext({
  user: null,
  theme: 'light',
  notifications: [],
  updateUser: () => {},
  updateTheme: () => {},
  addNotification: () => {},
});
```

```tsx
// GOOD - split contexts by concern
const UserContext = createContext<{
  user: User | null;
  updateUser: (user: User) => void;
} | null>(null);

const ThemeContext = createContext<{
  theme: 'light' | 'dark';
  updateTheme: (theme: 'light' | 'dark') => void;
} | null>(null);

const NotificationContext = createContext<{
  notifications: Notification[];
  addNotification: (notification: Notification) => void;
} | null>(null);

// Or use context with memoized values
const AppProvider = ({ children }: { children: React.ReactNode }) => {
  const [user, setUser] = useState<User | null>(null);
  const [theme, setTheme] = useState<'light' | 'dark'>('light');
  
  const userValue = useMemo(() => ({
    user,
    updateUser: setUser,
  }), [user]);
  
  const themeValue = useMemo(() => ({
    theme,
    updateTheme: setTheme,
  }), [theme]);
  
  return (
    <UserContext.Provider value={userValue}>
      <ThemeContext.Provider value={themeValue}>
        {children}
      </ThemeContext.Provider>
    </UserContext.Provider>
  );
};
```

Use proper key props for list items:

```tsx
// BAD - using array index as key
const TodoList = ({ todos }) => (
  <ul>
    {todos.map((todo, index) => (
      <li key={index}>{todo.text}</li>
    ))}
  </ul>
);
```

```tsx
// GOOD - using stable, unique keys
const TodoList = ({ todos }: { todos: Todo[] }) => (
  <ul>
    {todos.map(todo => (
      <li key={todo.id}>{todo.text}</li>
    ))}
  </ul>
);
```

Implement proper loading states and error boundaries:

```tsx
// GOOD - error boundary for performance monitoring
class ErrorBoundary extends React.Component<
  { children: React.ReactNode },
  { hasError: boolean }
> {
  constructor(props: { children: React.ReactNode }) {
    super(props);
    this.state = { hasError: false };
  }
  
  static getDerivedStateFromError(error: Error) {
    return { hasError: true };
  }
  
  componentDidCatch(error: Error, errorInfo: React.ErrorInfo) {
    console.error('Error caught by boundary:', error, errorInfo);
    // Log to error reporting service
  }
  
  render() {
    if (this.state.hasError) {
      return <div>Something went wrong. Please refresh the page.</div>;
    }
    
    return this.props.children;
  }
}

// Usage
const App = () => (
  <ErrorBoundary>
    <Router>
      <Routes>
        {/* Your routes */}
      </Routes>
    </Router>
  </ErrorBoundary>
);
```

Use React DevTools Profiler to identify performance bottlenecks:

```tsx
// GOOD - profiling wrapper for development
const ProfiledComponent = ({ children, id }: {
  children: React.ReactNode;
  id: string;
}) => {
  if (process.env.NODE_ENV === 'development') {
    return (
      <React.Profiler
        id={id}
        onRender={(id, phase, actualDuration) => {
          if (actualDuration > 16) { // More than one frame
            console.warn(`Slow render in ${id}: ${actualDuration}ms`);
          }
        }}
      >
        {children}
      </React.Profiler>
    );
  }
  
  return <>{children}</>;
};
```