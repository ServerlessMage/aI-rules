---
description: 
globs: 
alwaysApply: true
---
Ensure React applications are accessible to all users by following WCAG guidelines and using proper semantic HTML and ARIA attributes.

Use semantic HTML elements instead of generic divs:

```tsx
// BAD - generic elements without semantic meaning
const Navigation = () => (
  <div className="nav">
    <div className="nav-item">Home</div>
    <div className="nav-item">About</div>
    <div className="nav-item">Contact</div>
  </div>
);
```

```tsx
// GOOD - semantic HTML elements
const Navigation = () => (
  <nav aria-label="Main navigation">
    <ul>
      <li><a href="/">Home</a></li>
      <li><a href="/about">About</a></li>
      <li><a href="/contact">Contact</a></li>
    </ul>
  </nav>
);
```

Provide proper labels and descriptions for form elements:

```tsx
// BAD - missing labels and descriptions
const LoginForm = () => (
  <form>
    <input type="email" placeholder="Email" />
    <input type="password" placeholder="Password" />
    <button>Login</button>
  </form>
);
```

```tsx
// GOOD - proper labels and accessibility attributes
const LoginForm = () => {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [errors, setErrors] = useState<{ email?: string; password?: string }>({});
  
  return (
    <form aria-labelledby="login-heading">
      <h2 id="login-heading">Login to your account</h2>
      
      <div>
        <label htmlFor="email">Email address</label>
        <input
          id="email"
          type="email"
          value={email}
          onChange={(e) => setEmail(e.target.value)}
          aria-invalid={!!errors.email}
          aria-describedby={errors.email ? 'email-error' : 'email-help'}
          required
        />
        <div id="email-help">We'll never share your email with anyone else.</div>
        {errors.email && (
          <div id="email-error" role="alert" aria-live="polite">
            {errors.email}
          </div>
        )}
      </div>
      
      <div>
        <label htmlFor="password">Password</label>
        <input
          id="password"
          type="password"
          value={password}
          onChange={(e) => setPassword(e.target.value)}
          aria-invalid={!!errors.password}
          aria-describedby={errors.password ? 'password-error' : undefined}
          required
        />
        {errors.password && (
          <div id="password-error" role="alert" aria-live="polite">
            {errors.password}
          </div>
        )}
      </div>
      
      <button type="submit">Login</button>
    </form>
  );
};
```

Implement proper focus management:

```tsx
// GOOD - focus management in modal
const Modal = ({ isOpen, onClose, title, children }: {
  isOpen: boolean;
  onClose: () => void;
  title: string;
  children: React.ReactNode;
}) => {
  const modalRef = useRef<HTMLDivElement>(null);
  const previousFocusRef = useRef<HTMLElement | null>(null);
  
  useEffect(() => {
    if (isOpen) {
      // Store the previously focused element
      previousFocusRef.current = document.activeElement as HTMLElement;
      
      // Focus the modal
      modalRef.current?.focus();
      
      // Trap focus within modal
      const handleKeyDown = (event: KeyboardEvent) => {
        if (event.key === 'Escape') {
          onClose();
        }
        
        if (event.key === 'Tab') {
          const focusableElements = modalRef.current?.querySelectorAll(
            'button, [href], input, select, textarea, [tabindex]:not([tabindex="-1"])'
          );
          
          if (focusableElements && focusableElements.length > 0) {
            const firstElement = focusableElements[0] as HTMLElement;
            const lastElement = focusableElements[focusableElements.length - 1] as HTMLElement;
            
            if (event.shiftKey && document.activeElement === firstElement) {
              event.preventDefault();
              lastElement.focus();
            } else if (!event.shiftKey && document.activeElement === lastElement) {
              event.preventDefault();
              firstElement.focus();
            }
          }
        }
      };
      
      document.addEventListener('keydown', handleKeyDown);
      
      return () => {
        document.removeEventListener('keydown', handleKeyDown);
        // Restore focus to previously focused element
        previousFocusRef.current?.focus();
      };
    }
  }, [isOpen, onClose]);
  
  if (!isOpen) return null;
  
  return (
    <div className="modal-backdrop" onClick={(e) => e.target === e.currentTarget && onClose()}>
      <div
        ref={modalRef}
        className="modal-content"
        role="dialog"
        aria-modal="true"
        aria-labelledby="modal-title"
        tabIndex={-1}
      >
        <div className="modal-header">
          <h2 id="modal-title">{title}</h2>
          <button
            onClick={onClose}
            aria-label="Close modal"
            className="close-button"
          >
            ×
          </button>
        </div>
        <div className="modal-body">
          {children}
        </div>
      </div>
    </div>
  );
};
```

Use ARIA attributes for complex UI components:

```tsx
// GOOD - accessible dropdown with ARIA
const Dropdown = ({ label, options, value, onChange }: {
  label: string;
  options: Array<{ value: string; label: string }>;
  value: string;
  onChange: (value: string) => void;
}) => {
  const [isOpen, setIsOpen] = useState(false);
  const [focusedIndex, setFocusedIndex] = useState(-1);
  const dropdownRef = useRef<HTMLDivElement>(null);
  
  const handleKeyDown = (event: React.KeyboardEvent) => {
    switch (event.key) {
      case 'Enter':
      case ' ':
        event.preventDefault();
        if (focusedIndex >= 0) {
          onChange(options[focusedIndex].value);
          setIsOpen(false);
        } else {
          setIsOpen(!isOpen);
        }
        break;
      case 'Escape':
        setIsOpen(false);
        setFocusedIndex(-1);
        break;
      case 'ArrowDown':
        event.preventDefault();
        if (!isOpen) {
          setIsOpen(true);
        } else {
          setFocusedIndex(prev => 
            prev < options.length - 1 ? prev + 1 : 0
          );
        }
        break;
      case 'ArrowUp':
        event.preventDefault();
        if (isOpen) {
          setFocusedIndex(prev => 
            prev > 0 ? prev - 1 : options.length - 1
          );
        }
        break;
    }
  };
  
  return (
    <div className="dropdown" ref={dropdownRef}>
      <button
        type="button"
        aria-haspopup="listbox"
        aria-expanded={isOpen}
        aria-labelledby="dropdown-label"
        onClick={() => setIsOpen(!isOpen)}
        onKeyDown={handleKeyDown}
        className="dropdown-trigger"
      >
        <span id="dropdown-label">{label}</span>
        <span>{options.find(opt => opt.value === value)?.label || 'Select...'}</span>
      </button>
      
      {isOpen && (
        <ul
          role="listbox"
          aria-labelledby="dropdown-label"
          className="dropdown-menu"
        >
          {options.map((option, index) => (
            <li
              key={option.value}
              role="option"
              aria-selected={option.value === value}
              className={`dropdown-option ${
                index === focusedIndex ? 'focused' : ''
              } ${option.value === value ? 'selected' : ''}`}
              onClick={() => {
                onChange(option.value);
                setIsOpen(false);
              }}
            >
              {option.label}
            </li>
          ))}
        </ul>
      )}
    </div>
  );
};
```

Provide alternative text for images and media:

```tsx
// GOOD - proper alt text and media descriptions
const ProductCard = ({ product }: { product: Product }) => (
  <article className="product-card">
    <img
      src={product.image}
      alt={`${product.name} - ${product.description}`}
      loading="lazy"
    />
    <h3>{product.name}</h3>
    <p>{product.description}</p>
    <span aria-label={`Price: ${product.price} dollars`}>
      ${product.price}
    </span>
  </article>
);

// For decorative images
const DecorativeImage = () => (
  <img src="/decorative-pattern.png" alt="" role="presentation" />
);
```

Implement proper heading hierarchy:

```tsx
// GOOD - logical heading structure
const BlogPost = ({ post }: { post: BlogPost }) => (
  <article>
    <header>
      <h1>{post.title}</h1>
      <p>By {post.author} on {post.date}</p>
    </header>
    
    <div className="content">
      <h2>Introduction</h2>
      <p>{post.introduction}</p>
      
      <h2>Main Content</h2>
      <h3>Subsection 1</h3>
      <p>Content...</p>
      
      <h3>Subsection 2</h3>
      <p>Content...</p>
      
      <h2>Conclusion</h2>
      <p>{post.conclusion}</p>
    </div>
  </article>
);
```

Use live regions for dynamic content updates:

```tsx
// GOOD - live regions for status updates
const StatusMessage = ({ message, type }: {
  message: string;
  type: 'success' | 'error' | 'info';
}) => (
  <div
    role="alert"
    aria-live={type === 'error' ? 'assertive' : 'polite'}
    className={`status-message status-${type}`}
  >
    {message}
  </div>
);

const LoadingSpinner = ({ isLoading }: { isLoading: boolean }) => (
  <div
    aria-live="polite"
    aria-busy={isLoading}
    className="loading-container"
  >
    {isLoading ? (
      <>
        <div className="spinner" aria-hidden="true" />
        <span className="sr-only">Loading content...</span>
      </>
    ) : (
      <span className="sr-only">Content loaded</span>
    )}
  </div>
);
```

Test accessibility with automated tools and manual testing:

```tsx
// GOOD - accessibility testing utilities
const AccessibilityTestWrapper = ({ children }: { children: React.ReactNode }) => {
  useEffect(() => {
    if (process.env.NODE_ENV === 'development') {
      import('@axe-core/react').then(axe => {
        axe.default(React, ReactDOM, 1000);
      });
    }
  }, []);
  
  return <>{children}</>;
};

// Usage in development
const App = () => (
  <AccessibilityTestWrapper>
    <Router>
      {/* Your app content */}
    </Router>
  </AccessibilityTestWrapper>
);
```