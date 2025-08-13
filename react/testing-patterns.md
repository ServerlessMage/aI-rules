---
description: 
globs: 
alwaysApply: true
---
Write comprehensive tests for React components using React Testing Library and Jest, focusing on user behavior rather than implementation details.

Test components from the user's perspective:

```tsx
// BAD - testing implementation details
import { shallow } from 'enzyme';

test('Counter component', () => {
  const wrapper = shallow(<Counter />);
  expect(wrapper.state('count')).toBe(0);
  wrapper.instance().increment();
  expect(wrapper.state('count')).toBe(1);
});
```

```tsx
// GOOD - testing user interactions
import { render, screen, fireEvent } from '@testing-library/react';
import userEvent from '@testing-library/user-event';

test('Counter increments when button is clicked', async () => {
  const user = userEvent.setup();
  render(<Counter />);
  
  const counter = screen.getByText('Count: 0');
  const incrementButton = screen.getByRole('button', { name: /increment/i });
  
  await user.click(incrementButton);
  
  expect(screen.getByText('Count: 1')).toBeInTheDocument();
});
```

Use proper queries to find elements:

```tsx
// GOOD - accessibility-focused queries
test('Login form validation', async () => {
  const user = userEvent.setup();
  const mockSubmit = jest.fn();
  
  render(<LoginForm onSubmit={mockSubmit} />);
  
  // Use getByRole for interactive elements
  const emailInput = screen.getByRole('textbox', { name: /email/i });
  const passwordInput = screen.getByLabelText(/password/i);
  const submitButton = screen.getByRole('button', { name: /login/i });
  
  // Test validation
  await user.click(submitButton);
  
  expect(screen.getByText(/email is required/i)).toBeInTheDocument();
  expect(screen.getByText(/password is required/i)).toBeInTheDocument();
  expect(mockSubmit).not.toHaveBeenCalled();
  
  // Test successful submission
  await user.type(emailInput, 'user@example.com');
  await user.type(passwordInput, 'password123');
  await user.click(submitButton);
  
  expect(mockSubmit).toHaveBeenCalledWith({
    email: 'user@example.com',
    password: 'password123',
  });
});
```

Test async operations and loading states:

```tsx
// GOOD - testing async behavior
import { waitFor, waitForElementToBeRemoved } from '@testing-library/react';

test('UserProfile loads and displays user data', async () => {
  const mockUser = { id: '1', name: 'John Doe', email: 'john@example.com' };
  
  // Mock the API call
  jest.spyOn(api, 'getUser').mockResolvedValue(mockUser);
  
  render(<UserProfile userId="1" />);
  
  // Check loading state
  expect(screen.getByText(/loading/i)).toBeInTheDocument();
  
  // Wait for loading to finish
  await waitForElementToBeRemoved(() => screen.queryByText(/loading/i));
  
  // Check that user data is displayed
  expect(screen.getByText('John Doe')).toBeInTheDocument();
  expect(screen.getByText('john@example.com')).toBeInTheDocument();
  
  // Verify API was called correctly
  expect(api.getUser).toHaveBeenCalledWith('1');
});

test('UserProfile handles API errors', async () => {
  const consoleError = jest.spyOn(console, 'error').mockImplementation();
  
  // Mock API failure
  jest.spyOn(api, 'getUser').mockRejectedValue(new Error('User not found'));
  
  render(<UserProfile userId="1" />);
  
  await waitFor(() => {
    expect(screen.getByText(/error loading user/i)).toBeInTheDocument();
  });
  
  consoleError.mockRestore();
});
```

Test custom hooks:

```tsx
// GOOD - testing custom hooks
import { renderHook, act } from '@testing-library/react';

test('useCounter hook', () => {
  const { result } = renderHook(() => useCounter(0));
  
  expect(result.current.count).toBe(0);
  
  act(() => {
    result.current.increment();
  });
  
  expect(result.current.count).toBe(1);
  
  act(() => {
    result.current.decrement();
  });
  
  expect(result.current.count).toBe(0);
});

test('useLocalStorage hook', () => {
  const { result } = renderHook(() => useLocalStorage('test-key', 'initial'));
  
  expect(result.current[0]).toBe('initial');
  
  act(() => {
    result.current[1]('updated');
  });
  
  expect(result.current[0]).toBe('updated');
  expect(localStorage.getItem('test-key')).toBe('"updated"');
});
```

Test components with context:

```tsx
// GOOD - testing with context providers
const renderWithAuth = (ui: React.ReactElement, { user = null } = {}) => {
  const Wrapper = ({ children }: { children: React.ReactNode }) => (
    <AuthContext.Provider value={{ user, login: jest.fn(), logout: jest.fn() }}>
      {children}
    </AuthContext.Provider>
  );
  
  return render(ui, { wrapper: Wrapper });
};

test('Dashboard shows user-specific content when logged in', () => {
  const mockUser = { id: '1', name: 'John Doe', role: 'admin' };
  
  renderWithAuth(<Dashboard />, { user: mockUser });
  
  expect(screen.getByText('Welcome, John Doe')).toBeInTheDocument();
  expect(screen.getByText('Admin Panel')).toBeInTheDocument();
});

test('Dashboard shows login prompt when not logged in', () => {
  renderWithAuth(<Dashboard />);
  
  expect(screen.getByText('Please log in to continue')).toBeInTheDocument();
  expect(screen.queryByText('Admin Panel')).not.toBeInTheDocument();
});
```

Mock external dependencies properly:

```tsx
// GOOD - mocking external dependencies
jest.mock('../services/api', () => ({
  getUsers: jest.fn(),
  createUser: jest.fn(),
  updateUser: jest.fn(),
  deleteUser: jest.fn(),
}));

jest.mock('react-router-dom', () => ({
  ...jest.requireActual('react-router-dom'),
  useNavigate: () => jest.fn(),
  useParams: () => ({ id: '1' }),
}));

test('UserForm submits data correctly', async () => {
  const user = userEvent.setup();
  const mockNavigate = jest.fn();
  const mockCreateUser = api.createUser as jest.MockedFunction<typeof api.createUser>;
  
  mockCreateUser.mockResolvedValue({ id: '1', name: 'John Doe' });
  
  render(<UserForm />);
  
  await user.type(screen.getByLabelText(/name/i), 'John Doe');
  await user.type(screen.getByLabelText(/email/i), 'john@example.com');
  await user.click(screen.getByRole('button', { name: /save/i }));
  
  await waitFor(() => {
    expect(mockCreateUser).toHaveBeenCalledWith({
      name: 'John Doe',
      email: 'john@example.com',
    });
  });
});
```

Test accessibility in components:

```tsx
// GOOD - testing accessibility
import { axe, toHaveNoViolations } from 'jest-axe';

expect.extend(toHaveNoViolations);

test('LoginForm has no accessibility violations', async () => {
  const { container } = render(<LoginForm />);
  const results = await axe(container);
  expect(results).toHaveNoViolations();
});

test('Modal traps focus correctly', async () => {
  const user = userEvent.setup();
  
  render(<ModalExample />);
  
  const openButton = screen.getByRole('button', { name: /open modal/i });
  await user.click(openButton);
  
  const modal = screen.getByRole('dialog');
  const closeButton = screen.getByRole('button', { name: /close/i });
  
  expect(modal).toHaveFocus();
  
  // Test tab navigation
  await user.tab();
  expect(closeButton).toHaveFocus();
  
  // Test escape key
  await user.keyboard('{Escape}');
  expect(modal).not.toBeInTheDocument();
  expect(openButton).toHaveFocus();
});
```

Use test utilities and setup files:

```tsx
// test-utils.tsx - custom render function
import { render, RenderOptions } from '@testing-library/react';
import { BrowserRouter } from 'react-router-dom';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';

const AllTheProviders = ({ children }: { children: React.ReactNode }) => {
  const queryClient = new QueryClient({
    defaultOptions: {
      queries: { retry: false },
      mutations: { retry: false },
    },
  });
  
  return (
    <QueryClientProvider client={queryClient}>
      <BrowserRouter>
        {children}
      </BrowserRouter>
    </QueryClientProvider>
  );
};

const customRender = (ui: React.ReactElement, options?: RenderOptions) =>
  render(ui, { wrapper: AllTheProviders, ...options });

export * from '@testing-library/react';
export { customRender as render };
```

```tsx
// setupTests.ts
import '@testing-library/jest-dom';
import { server } from './mocks/server';

// Mock IntersectionObserver
global.IntersectionObserver = jest.fn().mockImplementation(() => ({
  observe: jest.fn(),
  unobserve: jest.fn(),
  disconnect: jest.fn(),
}));

// Setup MSW
beforeAll(() => server.listen());
afterEach(() => server.resetHandlers());
afterAll(() => server.close());
```