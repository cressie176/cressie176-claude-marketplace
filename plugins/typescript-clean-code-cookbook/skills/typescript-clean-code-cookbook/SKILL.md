---
name: typescript-clean-code-cookbook
description: Use this skill when writing, reviewing, or refactoring TypeScript code to ensure it follows clean code principles. Apply these standards to maintain clear, consistent, and maintainable code. Invoke automatically when generating functions, classes, or modules.
allowed-tools: Read, Grep, Glob, Edit
---

# TypeScript Clean Code Principles

This skill provides guidance on writing clean, maintainable TypeScript code. Apply these principles when writing new code, refactoring existing code, or reviewing code quality.

## Core Philosophy

- Keep code simple and easy to change
- Prefer readability over cleverness
- Always leave code cleaner than you found it
- Write code that reveals its intent through clear naming and structure

## Naming Conventions

**Apply clear, descriptive names that reveal intent:**

- Use descriptive names for variables, functions, classes, and files
- Avoid abbreviations except for:
  - Industry standard abbreviations
  - Defacto standard abbreviations
  - Common short forms: `id`, `tx`, `cb`, `err`
- Names must reveal the intent of the code
- Avoid repetition in names

**Exception:** Do not apply naming rules to names required by interfaces, inheritance hierarchies, or third-party APIs.

**Good:**
```typescript
function calculateMonthlyPayment(principal: number, interestRate: number): number {
  return principal * interestRate / 12;
}

const userRepository = new UserRepository();
const users = userRepository.findActive();
```

**Bad:**
```typescript
function calc(p: number, r: number): number {  // Unclear abbreviations
  return p * r / 12;
}

const repo = new UserRepository();
const u = repo.getActiveUsers();  // 'u' is not descriptive
```

**Good (concise when parameters reveal intent):**
```typescript
arguments.get(name);  // Not arguments.getArgument(name)
users.find(id);       // Not users.findUser(id)
```

## Type Safety

**Use explicit types and avoid `any`:**

- TypeScript's type system is your ally - use it fully
- Never use `any` except as an absolute last resort
- `any` disables type checking and defeats the purpose of using TypeScript
- Prefer `unknown` when the type is genuinely unknown - it forces you to narrow before use
- Define proper types or interfaces for all data structures
- Use generic types to maintain type safety in reusable code

**Exception:** `any` may be acceptable when integrating with poorly-typed third-party libraries, but immediately wrap it with properly typed code.

**Prefer explicit type definitions over inline typing:**

- Define types or interfaces separately for complex parameter types
- Inline types make function signatures unwieldy and hard to read
- Separate type definitions improve reusability and maintainability
- Use inline types only for very simple cases (primitives, single-property objects)

**Good (explicit type definitions):**
```typescript
interface CreateUserParams {
  name: string;
  email: string;
  age: number;
  preferences: {
    newsletter: boolean;
    notifications: boolean;
  };
}

function createUser(params: CreateUserParams): User {
  // Implementation
}

// Type is reusable and testable
const userData: CreateUserParams = {
  name: 'John',
  email: 'john@example.com',
  age: 30,
  preferences: {
    newsletter: true,
    notifications: false
  }
};
```

**Bad (inline typing makes signature unwieldy):**
```typescript
function createUser(params: {
  name: string;
  email: string;
  age: number;
  preferences: {
    newsletter: boolean;
    notifications: boolean;
  };
}): User {
  // Implementation
}

// Cannot reuse the type, harder to test
```

**Good (inline typing for simple cases):**
```typescript
function calculateTotal(items: { price: number }[]): number {
  return items.reduce((sum, item) => sum + item.price, 0);
}

function greet(user: { name: string }): string {
  return `Hello, ${user.name}`;
}
```

**Good (explicit types):**
```typescript
interface User {
  id: string;
  name: string;
  email: string;
  age: number;
}

function createUser(data: User): User {
  return { ...data };
}

function processData(value: unknown): string {
  if (typeof value === 'string') {
    return value.toUpperCase();
  }
  throw new Error('Expected string');
}
```

**Bad (using any):**
```typescript
function createUser(data: any): any {  // Lost all type safety
  return { ...data };
}

function processData(value: any): any {  // No type checking
  return value.toUpperCase();  // Runtime error if value isn't string
}
```

**Good (generic types maintain safety):**
```typescript
function findById<T extends { id: string }>(items: T[], id: string): T | undefined {
  return items.find(item => item.id === id);
}

const user = findById(users, '123');  // Type is User | undefined
const order = findById(orders, '456');  // Type is Order | undefined
```

**Bad (any loses type information):**
```typescript
function findById(items: any[], id: string): any {
  return items.find(item => item.id === id);
}

const user = findById(users, '123');  // Type is any - no safety
const order = findById(orders, '456');  // Type is any - no safety
```

**Good (wrapping third-party any):**
```typescript
import { poorlyTypedLibrary } from 'some-library';

interface LibraryResponse {
  data: string;
  status: number;
}

function callLibrary(input: string): LibraryResponse {
  const result: any = poorlyTypedLibrary.call(input);  // Last resort any
  // Immediately convert to proper types
  return {
    data: String(result.data),
    status: Number(result.status)
  };
}
```

## Function Design

**Write small, focused functions that do one thing well:**

- Function names must use verbs to describe the action
- Minimize parameters - ideally no more than two
- Avoid boolean parameters - they invite conditional logic
- Keep functions small: ideally no more than 8 meaningful lines of code
- Functions must not cause hidden side effects
- Operate at a single level of abstraction

**Single Level of Abstraction:**
- A function describes either *what* happens (high-level) or *how* it happens (low-level), never both
- Don't mix high-level orchestration with low-level implementation details
- If a function switches between abstraction levels, split it into separate functions

**Counting meaningful lines (for the 8-line guideline):**
- Exclude: function signature, blank lines, braces-only lines, comments, try/catch, logging calls
- Exclude: statements inlined into function calls or object/array literals
- Count: fluid APIs and chained calls as a single statement
- Count: array/object literal initialization as a single statement

**Exception:** Do not apply function rules to third-party library functions.

**Good (small, single purpose, verb name):**
```typescript
function validateEmail(email: string): boolean {
  const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
  return emailRegex.test(email);
}

function sendWelcomeEmail(user: User): void {
  const template = getWelcomeTemplate();
  const message = populateTemplate(template, user);
  emailService.send(user.email, message);
}
```

**Bad (too large, multiple responsibilities):**
```typescript
function processUser(user: User, sendEmail: boolean): void {  // Boolean parameter
  // Validation
  if (!user.email) throw new Error('No email');
  const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
  if (!emailRegex.test(user.email)) throw new Error('Invalid email');

  // Save to database
  const connection = createConnection();
  connection.query('INSERT INTO users...');

  // Send email conditionally
  if (sendEmail) {  // Conditional logic from boolean param
    const template = fs.readFileSync('welcome.html');
    const message = template.replace('{{name}}', user.name);
    smtp.send(user.email, message);
  }
}
```

**Good (single level of abstraction - high level):**
```typescript
function processOrder(order: Order): void {
  validateOrder(order);
  chargePayment(order);
  shipProducts(order);
  sendConfirmation(order);
}
```

**Bad (mixed abstraction levels):**
```typescript
function processOrder(order: Order): void {
  // High-level: validation
  validateOrder(order);

  // Low-level: payment processing details mixed in
  const stripe = new Stripe(apiKey);
  const charge = await stripe.charges.create({
    amount: order.total * 100,
    currency: 'usd',
    source: order.token
  });

  // High-level: shipping
  shipProducts(order);
}
```

## Async Functions

**Return promises directly - don't use `return await`:**

- When returning a promise from an async function, return it directly
- The caller has to await anyway, so awaiting inside the function is redundant and reduces the signal-to-noise raising of the code
- Only use `await` when you need the value for further processing

**Exception:** Use `return await` when inside a try-catch block and you need to catch errors from the promise.

**Good (return promise directly):**
```typescript
async function getUser(id: string): Promise<User> {
  return userRepository.findById(id);
}

async function processOrder(orderId: string): Promise<void> {
  return orderService.process(orderId);
}
```

**Bad (unnecessary await):**
```typescript
async function getUser(id: string): Promise<User> {
  return await userRepository.findById(id);  // Redundant await
}

async function processOrder(orderId: string): Promise<void> {
  return await orderService.process(orderId);  // Redundant await
}
```

**Good (await when you need the value):**
```typescript
async function getUserWithOrders(id: string): Promise<UserWithOrders> {
  const user = await userRepository.findById(id);  // Need user object
  const orders = await orderRepository.findByUserId(user.id);  // Use user.id

  return { user, orders };
}
```

**Exception - use await in try-catch:**
```typescript
async function getUser(id: string): Promise<User> {
  try {
    return await userRepository.findById(id);  // Needed to catch errors here
  } catch (error) {
    logger.error('Failed to fetch user', { id, error });
    throw error;
  }
}
```

## Conditional Logic

**Eliminate conditionals through design:**

- Avoid `else` blocks and nested conditionals
- Use guard clauses (early return) instead
- Do not use boolean parameters or boolean state variables
- Replace conditionals with:
  - Polymorphism (different classes implementing same interface)
  - Strategy/command objects (pass behaviour as objects)
  - Explicitly different functions (separate functions for separate paths)

**Move decisions upstream:**
- The caller (UI, API client, calling service) usually knows what outcome it wants
- If the caller is explicit, no conditional logic is needed
- Conditionals often indicate the caller delegated a decision it should have made

**Exception:** Do not apply these rules to code using the Factory pattern.

**Good (guard clauses):**
```typescript
function calculateDiscount(user: User, amount: number): number {
  if (!user.isPremium) return 0;
  if (amount < 100) return 0;

  return amount * 0.1;
}
```

**Bad (nested conditionals):**
```typescript
function calculateDiscount(user: User, amount: number): number {
  if (user.isPremium) {
    if (amount >= 100) {
      return amount * 0.1;
    } else {
      return 0;
    }
  } else {
    return 0;
  }
}
```

**Good (polymorphism instead of conditionals):**
```typescript
interface PaymentProcessor {
  process(amount: number): void;
}

class CreditCardProcessor implements PaymentProcessor {
  process(amount: number): void {
    // Credit card specific logic
  }
}

class PayPalProcessor implements PaymentProcessor {
  process(amount: number): void {
    // PayPal specific logic
  }
}

function checkout(processor: PaymentProcessor, amount: number): void {
  processor.process(amount);  // No conditionals
}
```

**Bad (conditional logic):**
```typescript
function checkout(paymentType: string, amount: number): void {
  if (paymentType === 'credit-card') {
    // Credit card logic
  } else if (paymentType === 'paypal') {
    // PayPal logic
  } else if (paymentType === 'bitcoin') {
    // Bitcoin logic
  }
}
```

**Good (separate functions instead of boolean parameter):**
```typescript
function activateUser(user: User): void {
  user.status = 'active';
  sendActivationEmail(user);
}

function deactivateUser(user: User): void {
  user.status = 'inactive';
  sendDeactivationEmail(user);
}
```

**Bad (boolean parameter creates conditional):**
```typescript
function setUserStatus(user: User, activate: boolean): void {
  if (activate) {
    user.status = 'active';
    sendActivationEmail(user);
  } else {
    user.status = 'inactive';
    sendDeactivationEmail(user);
  }
}
```

## Comments

**Comments should explain *why*, never *what*:**

- Allow comments only when they add genuine value
- Comments explain why code exists, not what it does
- Prefer self-explanatory code over comments
- Code should be readable enough that comments aren't needed

**Valid reasons for comments:**
- Referencing a bug or issue
- Explaining something surprising or counterintuitive
- Documenting code standards violations with justification

**Good:**
```typescript
// Using setTimeout instead of setInterval because the API rate limit
// requires we wait for each response before making the next request
function pollApi(): void {
  fetchData().then(() => {
    setTimeout(pollApi, 1000);
  });
}

// Workaround for bug in library v2.3.1 - see issue #1234
const result = workaroundFunction();
```

**Bad:**
```typescript
// Loop through users
users.forEach(user => {
  // Check if user is active
  if (user.isActive) {
    // Send email
    sendEmail(user);
  }
});

// Increment counter
counter++;
```

**Better (no comments needed):**
```typescript
activeUsers.forEach(sendEmail);

counter++;
```

## Objects and Data Structures

**Practice proper encapsulation:**

- Hide implementation details inside objects
- Expose behaviour through clear methods, never raw data
- Avoid public accessor (getter) methods
- Avoid public setter methods
- Provide meaningful behaviour methods that perform actions without exposing internals

**Avoid helpers and utils:**
- Helper/util methods indicate behaviour without a natural home
- This signals an incomplete or incorrectly designed domain model
- Move behaviour into the object that owns the data
- Create new domain objects to represent missing concepts

**Exception:** Do not apply these rules to code whose sole purpose is to act as a conduit for configuration or request data (e.g., DTOs, configuration objects).

**Good (behaviour, not data exposure):**
```typescript
class ShoppingCart {
  private items: Item[] = [];

  addItem(item: Item): void {
    this.items.push(item);
  }

  calculateTotal(): number {
    return this.items.reduce((sum, item) => sum + item.price, 0);
  }

  applyDiscount(percentage: number): void {
    this.items.forEach(item => {
      item.price *= (1 - percentage / 100);
    });
  }
}
```

**Bad (exposed data, separate helper):**
```typescript
class ShoppingCart {
  public items: Item[] = [];  // Exposed internal data

  getItems(): Item[] {  // Unnecessary getter
    return this.items;
  }

  setItems(items: Item[]): void {  // Unnecessary setter
    this.items = items;
  }
}

// Helper function indicates missing domain concept
function calculateCartTotal(cart: ShoppingCart): number {
  return cart.getItems().reduce((sum, item) => sum + item.price, 0);
}

// Another helper - behaviour should be in the cart
function applyDiscountToCart(cart: ShoppingCart, percentage: number): void {
  cart.getItems().forEach(item => {
    item.price *= (1 - percentage / 100);
  });
}
```

## Error Handling

**Fail fast and fail clearly:**

- Use exceptions, never return codes
- Always provide meaningful error messages
- Stop execution immediately when invalid state is detected
- Don't hide errors or continue with corrupted state

**Good:**
```typescript
function processPayment(amount: number): void {
  if (amount <= 0) {
    throw new Error(`Invalid payment amount: ${amount}. Amount must be positive.`);
  }

  if (!isConnectedToPaymentGateway()) {
    throw new Error('Cannot process payment: payment gateway unavailable');
  }

  // Process payment
}
```

**Bad:**
```typescript
function processPayment(amount: number): number {
  if (amount <= 0) {
    return -1;  // Return code instead of exception
  }

  if (!isConnectedToPaymentGateway()) {
    console.log('Warning: gateway unavailable');
    return 0;  // Hiding error, continuing with corrupted state
  }

  // Process payment
  return 1;  // Success code
}
```

## DRY (Don't Repeat Yourself)

**Eliminate duplication ruthlessly:**

- Never duplicate logic or structure
- Extract repeated code into a single function or module
- Duplication makes maintenance harder and introduces bugs
- If you see the same pattern twice, abstract it

**Good:**
```typescript
function validateUserInput(input: string, fieldName: string): void {
  if (!input || input.trim().length === 0) {
    throw new Error(`${fieldName} is required`);
  }
}

function createUser(name: string, email: string, password: string): User {
  validateUserInput(name, 'Name');
  validateUserInput(email, 'Email');
  validateUserInput(password, 'Password');

  return new User(name, email, password);
}
```

**Bad:**
```typescript
function createUser(name: string, email: string, password: string): User {
  // Repeated validation logic
  if (!name || name.trim().length === 0) {
    throw new Error('Name is required');
  }

  if (!email || email.trim().length === 0) {
    throw new Error('Email is required');
  }

  if (!password || password.trim().length === 0) {
    throw new Error('Password is required');
  }

  return new User(name, email, password);
}
```
