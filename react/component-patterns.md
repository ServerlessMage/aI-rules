---
description: 
globs: 
alwaysApply: true
---
Use functional components with hooks instead of class components. Keep components small, focused, and properly typed.

```tsx
// BAD - class component
class UserProfile extends React.Component {
  constructor(props) {
    super(props);
    this.state = { loading: true };
  }
  
  componentDidMount() {
    this.fetchUser();
  }
  
  render() {
    return <div>{this.props.user.name}</div>;
  }
}
```

```tsx
// GOOD - functional component with hooks
interface UserProfileProps {
  readonly userId: string;
}

const UserProfile = ({ userId }: UserProfileProps) => {
  const [loading, setLoading] = useState(true);
  const [user, setUser] = useState<User | null>(null);
  
  useEffect(() => {
    fetchUser(userId).then(setUser).finally(() => setLoading(false));
  }, [userId]);
  
  if (loading) return <div>Loading...</div>;
  if (!user) return <div>User not found</div>;
  
  return <div>{user.name}</div>;
};
```

Use proper component composition and avoid prop drilling:

```tsx
// BAD - prop drilling
const App = () => {
  const user = useUser();
  return <Header user={user} />;
};

const Header = ({ user }) => <Navigation user={user} />;
const Navigation = ({ user }) => <UserMenu user={user} />;
```

```tsx
// GOOD - composition and context
const UserContext = createContext<User | null>(null);

const App = () => {
  const user = useUser();
  return (
    <UserContext.Provider value={user}>
      <Header />
    </UserContext.Provider>
  );
};

const Header = () => <Navigation />;
const Navigation = () => <UserMenu />;
const UserMenu = () => {
  const user = useContext(UserContext);
  return <div>{user?.name}</div>;
};
```

Use compound components for related UI elements:

```tsx
// GOOD - compound component pattern
const Card = ({ children }: { children: React.ReactNode }) => (
  <div className="card">{children}</div>
);

const CardHeader = ({ children }: { children: React.ReactNode }) => (
  <div className="card-header">{children}</div>
);

const CardBody = ({ children }: { children: React.ReactNode }) => (
  <div className="card-body">{children}</div>
);

Card.Header = CardHeader;
Card.Body = CardBody;

// Usage
const UserCard = () => (
  <Card>
    <Card.Header>User Profile</Card.Header>
    <Card.Body>User details here</Card.Body>
  </Card>
);
```

Use render props or custom hooks for logic reuse:

```tsx
// GOOD - custom hook for reusable logic
const useApi = <T>(url: string) => {
  const [data, setData] = useState<T | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<Error | null>(null);
  
  useEffect(() => {
    fetch(url)
      .then(res => res.json())
      .then(setData)
      .catch(setError)
      .finally(() => setLoading(false));
  }, [url]);
  
  return { data, loading, error };
};

// Usage in components
const UserList = () => {
  const { data: users, loading, error } = useApi<User[]>('/api/users');
  
  if (loading) return <div>Loading...</div>;
  if (error) return <div>Error: {error.message}</div>;
  
  return (
    <ul>
      {users?.map(user => (
        <li key={user.id}>{user.name}</li>
      ))}
    </ul>
  );
};
```

Always provide proper TypeScript interfaces for props:

```tsx
// GOOD - comprehensive prop types
interface ButtonProps {
  readonly children: React.ReactNode;
  readonly variant?: 'primary' | 'secondary' | 'danger';
  readonly size?: 'small' | 'medium' | 'large';
  readonly disabled?: boolean;
  readonly loading?: boolean;
  readonly onClick?: (event: React.MouseEvent<HTMLButtonElement>) => void;
  readonly className?: string;
  readonly type?: 'button' | 'submit' | 'reset';
}

const Button = ({
  children,
  variant = 'primary',
  size = 'medium',
  disabled = false,
  loading = false,
  onClick,
  className = '',
  type = 'button',
}: ButtonProps) => {
  return (
    <button
      type={type}
      disabled={disabled || loading}
      onClick={onClick}
      className={`btn btn-${variant} btn-${size} ${className}`}
    >
      {loading ? 'Loading...' : children}
    </button>
  );
};
```