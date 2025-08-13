---
description: 
globs: 
alwaysApply: true
---
Handle events properly with correct TypeScript types and prevent common pitfalls like memory leaks and performance issues.

Use proper TypeScript event types:

```tsx
// BAD - generic event type
const handleClick = (event: any) => {
  event.preventDefault();
};
```

```tsx
// GOOD - specific event types
const handleClick = (event: React.MouseEvent<HTMLButtonElement>) => {
  event.preventDefault();
  console.log('Button clicked');
};

const handleChange = (event: React.ChangeEvent<HTMLInputElement>) => {
  const value = event.target.value;
  setValue(value);
};

const handleSubmit = (event: React.FormEvent<HTMLFormElement>) => {
  event.preventDefault();
  const formData = new FormData(event.currentTarget);
  // Process form data
};
```

Use useCallback for event handlers to prevent unnecessary re-renders:

```tsx
// BAD - recreating handler on every render
const TodoItem = ({ todo, onToggle }: { todo: Todo; onToggle: (id: string) => void }) => {
  return (
    <li>
      <input
        type="checkbox"
        checked={todo.completed}
        onChange={() => onToggle(todo.id)} // New function every render
      />
      {todo.text}
    </li>
  );
};
```

```tsx
// GOOD - memoized event handler
const TodoItem = ({ todo, onToggle }: { todo: Todo; onToggle: (id: string) => void }) => {
  const handleToggle = useCallback(() => {
    onToggle(todo.id);
  }, [todo.id, onToggle]);
  
  return (
    <li>
      <input
        type="checkbox"
        checked={todo.completed}
        onChange={handleToggle}
      />
      {todo.text}
    </li>
  );
};
```

Handle form events properly:

```tsx
// GOOD - comprehensive form handling
interface FormData {
  readonly name: string;
  readonly email: string;
  readonly message: string;
}

const ContactForm = () => {
  const [formData, setFormData] = useState<FormData>({
    name: '',
    email: '',
    message: '',
  });
  const [errors, setErrors] = useState<Partial<FormData>>({});
  const [isSubmitting, setIsSubmitting] = useState(false);
  
  const handleInputChange = (event: React.ChangeEvent<HTMLInputElement | HTMLTextAreaElement>) => {
    const { name, value } = event.target;
    setFormData(prev => ({ ...prev, [name]: value }));
    
    // Clear error when user starts typing
    if (errors[name as keyof FormData]) {
      setErrors(prev => ({ ...prev, [name]: undefined }));
    }
  };
  
  const validateForm = (): boolean => {
    const newErrors: Partial<FormData> = {};
    
    if (!formData.name.trim()) {
      newErrors.name = 'Name is required';
    }
    
    if (!formData.email.trim()) {
      newErrors.email = 'Email is required';
    } else if (!/\S+@\S+\.\S+/.test(formData.email)) {
      newErrors.email = 'Email is invalid';
    }
    
    if (!formData.message.trim()) {
      newErrors.message = 'Message is required';
    }
    
    setErrors(newErrors);
    return Object.keys(newErrors).length === 0;
  };
  
  const handleSubmit = async (event: React.FormEvent<HTMLFormElement>) => {
    event.preventDefault();
    
    if (!validateForm()) return;
    
    setIsSubmitting(true);
    try {
      await submitForm(formData);
      setFormData({ name: '', email: '', message: '' });
    } catch (error) {
      console.error('Form submission failed:', error);
    } finally {
      setIsSubmitting(false);
    }
  };
  
  return (
    <form onSubmit={handleSubmit}>
      <div>
        <input
          type="text"
          name="name"
          value={formData.name}
          onChange={handleInputChange}
          placeholder="Name"
          aria-invalid={!!errors.name}
          aria-describedby={errors.name ? 'name-error' : undefined}
        />
        {errors.name && <span id="name-error" role="alert">{errors.name}</span>}
      </div>
      
      <div>
        <input
          type="email"
          name="email"
          value={formData.email}
          onChange={handleInputChange}
          placeholder="Email"
          aria-invalid={!!errors.email}
          aria-describedby={errors.email ? 'email-error' : undefined}
        />
        {errors.email && <span id="email-error" role="alert">{errors.email}</span>}
      </div>
      
      <div>
        <textarea
          name="message"
          value={formData.message}
          onChange={handleInputChange}
          placeholder="Message"
          aria-invalid={!!errors.message}
          aria-describedby={errors.message ? 'message-error' : undefined}
        />
        {errors.message && <span id="message-error" role="alert">{errors.message}</span>}
      </div>
      
      <button type="submit" disabled={isSubmitting}>
        {isSubmitting ? 'Submitting...' : 'Submit'}
      </button>
    </form>
  );
};
```

Handle keyboard events for accessibility:

```tsx
// GOOD - keyboard event handling
const Modal = ({ isOpen, onClose, children }: {
  isOpen: boolean;
  onClose: () => void;
  children: React.ReactNode;
}) => {
  useEffect(() => {
    const handleKeyDown = (event: KeyboardEvent) => {
      if (event.key === 'Escape') {
        onClose();
      }
    };
    
    if (isOpen) {
      document.addEventListener('keydown', handleKeyDown);
      return () => document.removeEventListener('keydown', handleKeyDown);
    }
  }, [isOpen, onClose]);
  
  const handleBackdropClick = (event: React.MouseEvent<HTMLDivElement>) => {
    if (event.target === event.currentTarget) {
      onClose();
    }
  };
  
  if (!isOpen) return null;
  
  return (
    <div className="modal-backdrop" onClick={handleBackdropClick}>
      <div className="modal-content" role="dialog" aria-modal="true">
        {children}
      </div>
    </div>
  );
};
```

Clean up event listeners to prevent memory leaks:

```tsx
// GOOD - proper cleanup
const WindowSizeTracker = () => {
  const [windowSize, setWindowSize] = useState({
    width: window.innerWidth,
    height: window.innerHeight,
  });
  
  useEffect(() => {
    const handleResize = () => {
      setWindowSize({
        width: window.innerWidth,
        height: window.innerHeight,
      });
    };
    
    window.addEventListener('resize', handleResize);
    
    // Cleanup function
    return () => {
      window.removeEventListener('resize', handleResize);
    };
  }, []);
  
  return <div>Window size: {windowSize.width} x {windowSize.height}</div>;
};
```

Use event delegation for dynamic lists:

```tsx
// GOOD - event delegation
const TodoList = ({ todos, onToggle, onDelete }: {
  todos: Todo[];
  onToggle: (id: string) => void;
  onDelete: (id: string) => void;
}) => {
  const handleListClick = (event: React.MouseEvent<HTMLUListElement>) => {
    const target = event.target as HTMLElement;
    const todoItem = target.closest('[data-todo-id]') as HTMLElement;
    
    if (!todoItem) return;
    
    const todoId = todoItem.dataset.todoId!;
    
    if (target.matches('input[type="checkbox"]')) {
      onToggle(todoId);
    } else if (target.matches('.delete-button')) {
      onDelete(todoId);
    }
  };
  
  return (
    <ul onClick={handleListClick}>
      {todos.map(todo => (
        <li key={todo.id} data-todo-id={todo.id}>
          <input type="checkbox" checked={todo.completed} readOnly />
          <span>{todo.text}</span>
          <button className="delete-button">Delete</button>
        </li>
      ))}
    </ul>
  );
};
```