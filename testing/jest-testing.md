---
description: 
globs: 
alwaysApply: false
---
## Fundamentals

**Organize tests by feature or module:**
- Group tests into files or directories that correspond to the features or modules they are testing
- Create a `__tests__` directory alongside your source code files or use `.test.js`/`.spec.js` suffixes
- Keep test files close to the code they test for better maintainability

```
src/
  components/
    UserProfile/
      UserProfile.tsx
      UserProfile.test.tsx
  services/
    __tests__/
      userService.test.ts
```

**Use descriptive test names and logical grouping:**
- Write test names that clearly describe what is being verified
- Use `describe` and `it` blocks to structure tests and provide context
- Use nested `describe` blocks for complex scenarios

```ts
describe('UserService', () => {
  describe('createUser', () => {
    it('should create a user with valid data', () => {
      // Test implementation
    });

    it('should throw error when email is invalid', () => {
      // Test implementation
    });
  });
});
```

**Keep tests isolated and independent:**
- Each test should run independently without shared state
- Use `beforeEach` and `afterEach` hooks for setup and cleanup
- Avoid dependencies between tests

```ts
describe('Calculator', () => {
  let calculator: Calculator;

  beforeEach(() => {
    calculator = new Calculator();
  });

  it('should add two numbers correctly', () => {
    expect(calculator.add(2, 3)).toBe(5);
  });
});
```

## Core Testing Principles

**Focus on behavior, not implementation:**
- Test the public API and expected outcomes
- Avoid testing internal implementation details
- Test the "what", not the "how"

```ts
// Good - tests behavior
it('should return user data when valid ID is provided', () => {
  const user = getUserById('123');
  expect(user).toHaveProperty('name');
  expect(user).toHaveProperty('email');
});

// Bad - tests implementation
it('should call database.query with correct SQL', () => {
  getUserById('123');
  expect(database.query).toHaveBeenCalledWith('SELECT * FROM users WHERE id = ?', ['123']);
});
```

**Use Jest's built-in matchers effectively:**
- Use appropriate matchers for different assertions
- Explore Jest's rich set of matchers for specific conditions

```ts
// Primitive values and identity
expect(result).toBe(42);
expect(user).toEqual({ id: 1, name: 'John' });

// Collections and properties
expect(users).toHaveLength(3);
expect(response).toHaveProperty('data.users');
expect(tags).toContain('javascript');

// Strings and patterns
expect(message).toMatch(/error/i);
expect(email).toMatch(/^[\w-\.]+@([\w-]+\.)+[\w-]{2,4}$/);

// Truthiness
expect(value).toBeTruthy();
expect(emptyString).toBeFalsy();
```

## Asynchronous Testing

**Handle asynchronous code correctly:**
- Use `async/await` or Promises to handle asynchronous code in tests
- Use `resolves` and `rejects` matchers for Promise assertions

```ts
// Using async/await
it('should fetch user data successfully', async () => {
  const user = await fetchUser(1);
  expect(user.name).toBe('John');
});

it('should handle fetch errors', async () => {
  await expect(fetchUser(-1)).rejects.toThrow('User not found');
});

// Using promise matchers
expect(fetchUser(1)).resolves.toHaveProperty('name');
expect(fetchUser(-1)).rejects.toThrow();
```

**Test concurrent operations and complex async patterns:**
```ts
it('should handle concurrent API calls', async () => {
  const promises = [fetchUser('1'), fetchUser('2'), fetchUser('3')];
  const users = await Promise.all(promises);
  
  expect(users).toHaveLength(3);
  expect(users.every(user => user.id)).toBe(true);
});

it('should handle race conditions', async () => {
  const slowPromise = new Promise(resolve => setTimeout(() => resolve('slow'), 100));
  const fastPromise = Promise.resolve('fast');
  
  const result = await Promise.race([slowPromise, fastPromise]);
  expect(result).toBe('fast');
});
```

## Error Handling and Edge Cases

**Test error conditions thoroughly:**
- Verify that functions throw appropriate errors
- Test edge cases and boundary conditions
- Include both happy path and error scenarios

```ts
describe('validateEmail', () => {
  it('should accept valid email addresses', () => {
    expect(validateEmail('user@example.com')).toBe(true);
  });

  it('should reject invalid email formats', () => {
    expect(() => validateEmail('invalid-email')).toThrow('Invalid email format');
    expect(() => validateEmail('')).toThrow('Email is required');
  });

  it('should handle edge cases', () => {
    expect(() => validateEmail(null)).toThrow();
    expect(() => validateEmail(undefined)).toThrow();
  });
});
```

## Mocking and Dependencies

**Mock external dependencies effectively:**
- Use mocking to isolate code from external dependencies
- Use `jest.mock` for module mocking and `jest.spyOn` for method spying

```ts
// Basic module mocking
jest.mock('./userService');
const mockUserService = userService as jest.Mocked<typeof userService>;

// Partial module mock
jest.mock('./userService', () => ({
  ...jest.requireActual('./userService'),
  fetchUser: jest.fn(),
}));

// Spy on specific methods
const fetchUserSpy = jest.spyOn(userService, 'fetchUser');
```

**Advanced mocking strategies:**
```ts
// Mock HTTP requests
jest.mock('axios');
const mockedAxios = axios as jest.Mocked<typeof axios>;

beforeEach(() => {
  mockedAxios.get.mockResolvedValue({
    data: { id: 1, name: 'John' },
    status: 200,
  });
});

// Mock with different return values per call
const mockFn = jest.fn()
  .mockReturnValueOnce('first call')
  .mockReturnValueOnce('second call')
  .mockReturnValue('default');

// Mock with conditional logic
const mockUserService = {
  fetchUser: jest.fn().mockImplementation((id: string) => {
    if (id === 'invalid') {
      throw new Error('User not found');
    }
    return Promise.resolve({ id, name: `User ${id}` });
  }),
};
```

## Setup and Teardown

**Use lifecycle hooks appropriately:**
- Use `beforeAll` and `afterAll` for expensive setup/teardown
- Use `beforeEach` and `afterEach` for test-specific setup/teardown

```ts
describe('Database operations', () => {
  beforeAll(async () => {
    // Expensive setup - run once
    await database.connect();
  });

  afterAll(async () => {
    // Cleanup - run once
    await database.disconnect();
  });

  beforeEach(() => {
    // Reset state before each test
    database.clearTables();
    jest.clearAllMocks();
  });

  it('should save user data', () => {
    // Test implementation
  });
});
```

## Test Data Management

**Keep test data clean and maintainable:**
- Externalize test data to improve readability and maintainability
- Use factories or builders for creating test data
- Avoid hardcoding data directly in tests

```ts
// Test data factory
const createTestUser = (overrides = {}) => ({
  id: 1,
  name: 'John Doe',
  email: 'john@example.com',
  ...overrides,
});

// Builder pattern for complex objects
class UserBuilder {
  private user: Partial<User> = {};

  withId(id: string): UserBuilder {
    this.user.id = id;
    return this;
  }

  withEmail(email: string): UserBuilder {
    this.user.email = email;
    return this;
  }

  build(): User {
    return {
      id: this.user.id || 'default-id',
      email: this.user.email || 'test@example.com',
      ...this.user,
    } as User;
  }
}

it('should handle admin users', () => {
  const admin = new UserBuilder()
    .withRole('admin')
    .withEmail('admin@company.com')
    .build();
    
  expect(canAccessAdminPanel(admin)).toBe(true);
});
```

## Advanced Testing Patterns

**Parameterized tests:**
```ts
describe.each([
  { input: 'valid@email.com', expected: true },
  { input: 'invalid-email', expected: false },
  { input: '', expected: false },
  { input: null, expected: false },
])('Email validation', ({ input, expected }) => {
  it(`should return ${expected} for input: ${input}`, () => {
    expect(isValidEmail(input)).toBe(expected);
  });
});

// Or with test.each for individual tests
test.each([
  [1, 2, 3],
  [2, 3, 5],
  [3, 4, 7],
])('add(%i, %i) should equal %i', (a, b, expected) => {
  expect(add(a, b)).toBe(expected);
});
```

**Testing timers and time-dependent code:**
```ts
describe('Timer-dependent functionality', () => {
  beforeEach(() => {
    jest.useFakeTimers();
  });

  afterEach(() => {
    jest.useRealTimers();
  });

  it('should execute callback after delay', () => {
    const callback = jest.fn();
    setTimeout(callback, 1000);

    expect(callback).not.toHaveBeenCalled();
    
    jest.advanceTimersByTime(1000);
    expect(callback).toHaveBeenCalledTimes(1);
  });
});
```

## React Component Testing

**Test component integration and state:**
```ts
import { render, screen, fireEvent, waitFor } from '@testing-library/react';

const renderWithProvider = (component: React.ReactElement, user = mockUser) => {
  return render(
    <UserProvider value={{ user, updateUser: jest.fn() }}>
      {component}
    </UserProvider>
  );
};

it('should update user profile on form submission', async () => {
  const mockUpdateUser = jest.fn();
  renderWithProvider(<UserProfile />, { ...mockUser, updateUser: mockUpdateUser });

  fireEvent.change(screen.getByLabelText(/name/i), {
    target: { value: 'New Name' },
  });
  
  fireEvent.click(screen.getByRole('button', { name: /save/i }));

  await waitFor(() => {
    expect(mockUpdateUser).toHaveBeenCalledWith({ ...mockUser, name: 'New Name' });
  });
});
```

**Test custom hooks:**
```ts
import { renderHook, act } from '@testing-library/react';

it('should manage counter state correctly', () => {
  const { result } = renderHook(() => useCounter(0));

  expect(result.current.count).toBe(0);

  act(() => {
    result.current.increment();
  });

  expect(result.current.count).toBe(1);
});
```

## Performance and Optimization

**Optimize test performance:**
- Use `jest.clearAllMocks()` and `jest.resetAllMocks()` in `beforeEach` blocks to prevent state leakage
- Avoid creating large data structures in tests that are not necessary
- Mock expensive operations to speed up tests
- Use `beforeAll` for expensive setup that can be shared

```ts
describe('Performance-optimized tests', () => {
  beforeEach(() => {
    jest.clearAllMocks();
  });

  beforeAll(async () => {
    await setupDatabase(); // Expensive setup once
  });

  it('should not perform expensive calculations in tests', () => {
    const expensiveOperation = jest.fn().mockReturnValue('mocked result');
    const result = processData(expensiveOperation);
    expect(result).toBeDefined();
    expect(expensiveOperation).toHaveBeenCalledTimes(1);
  });
});
```

## Configuration and Environment

**Configure Jest correctly:**
- Configure Jest using `jest.config.js` or `jest.config.ts` file
- Configure settings such as test environment, module file extensions, and coverage thresholds
- Consider using presets like `ts-jest` or `babel-jest` for TypeScript or Babel support

```ts
// jest.config.js
module.exports = {
  testEnvironment: 'jsdom',
  setupFilesAfterEnv: ['<rootDir>/src/test/setup.ts'],
  moduleNameMapping: {
    '^@/(.*)$': '<rootDir>/src/$1',
  },
  collectCoverageFrom: [
    'src/**/*.{ts,tsx}',
    '!src/**/*.d.ts',
    '!src/test/**',
  ],
  coverageThreshold: {
    global: {
      branches: 80,
      functions: 80,
      lines: 80,
      statements: 80,
    },
  },
};
```

**Environment-specific testing:**
```ts
describe('Environment-specific behavior', () => {
  const originalNodeEnv = process.env.NODE_ENV;

  afterEach(() => {
    process.env.NODE_ENV = originalNodeEnv;
  });

  it('should behave differently in production', () => {
    process.env.NODE_ENV = 'production';
    
    const result = getConfigForEnvironment();
    expect(result.debug).toBe(false);
    expect(result.logLevel).toBe('error');
  });
});
```

## Best Practices and Common Pitfalls

**Write tests that are easy to read and maintain:**
- Keep tests concise and focused
- Use clear and consistent formatting
- Add comments to explain complex logic
- Refactor tests regularly to keep them up to date

**Aim for high test coverage, but prioritize meaningful tests:**
- Aim for high test coverage to ensure code is well-tested
- Prioritize writing meaningful tests that verify core functionality
- Don't aim for 100% coverage without considering the value of each test
- Use coverage reports to identify areas that need more testing

**Use snapshots sparingly:**
- Snapshots can be useful for verifying component output
- However, they can be brittle and difficult to maintain if used excessively
- Use snapshots strategically and review them carefully when they change

**Common mistakes to avoid:**
- Forgetting to mock dependencies
- Testing implementation details instead of behavior
- Creating brittle snapshots
- Relying on global state
- Not testing error conditions
- Creating tests that depend on each other

**Debugging strategies:**
- Use `console.log` statements for quick debugging
- Use Jest's interactive mode (`--watch`) for development
- Use debuggers when needed
- Check mock call history with `toHaveBeenCalledWith`

## Integration with Development Workflow

**Run tests frequently:**
- Run tests frequently to catch errors early
- Use Jest's watch mode (`--watch`) to automatically run tests when files change
- Integrate tests into your CI/CD pipeline

**Tooling and environment:**
- Use recommended development tools: VS Code, WebStorm, Jest Runner extension
- Configure linting and formatting with ESLint and Prettier
- Integrate with CI/CD tools like GitHub Actions, Jenkins, or CircleCI

**Consider test-driven development (TDD):**
- Write tests before writing code to drive the development process
- This can help write more testable and well-designed code
- Use the red-green-refactor cycle for systematic development
