# Stagehand: From English Instructions to Browser Automation

## Overview

Stagehand bridges natural language instructions and browser automation by using AI to understand semantic page structure through accessibility trees, then translating that understanding into precise Playwright commands.

## The Three Core Methods

### 1. `act()` - Direct Action Execution
```typescript
await page.act("Click the search box");
```
- **Purpose**: Execute a single action immediately
- **When to use**: Simple, unambiguous actions where you trust the AI to find the right element
- **Behind the scenes**: Combines observe + execute in one call

### 2. `observe()` - Plan Before Acting
```typescript
const [action] = await page.observe("Find the submit button");
await page.act(action);
```
- **Purpose**: Get element details before acting, allowing inspection or modification
- **When to use**: When you want to verify the target or need more control
- **Returns**: Array of action objects with element details and methods

#### Why use `observe()` when `act()` calls it internally?

1. **Performance & Caching**: Save money and time by caching observations
   ```typescript
   // Cache the observation
   const [submitButton] = await page.observe("Find the submit button");
   
   // Reuse it multiple times without additional LLM calls
   await page.act(submitButton); // No LLM call!
   await page.act(submitButton); // Still no LLM call!
   ```

2. **Sensitive Data Protection**: Keep passwords and personal info away from LLMs
   ```typescript
   const [passwordField] = await page.observe("Find the password field");
   // Modify locally without sending sensitive data to AI
   passwordField.arguments = [process.env.SECRET_PASSWORD];
   await page.act(passwordField);
   ```

3. **Multiple Elements**: See all matching elements, not just the first
   ```typescript
   const allButtons = await page.observe("Find all download buttons");
   console.log(`Found ${allButtons.length} download buttons`);
   // Choose which one to click based on your logic
   ```

4. **Conditional Actions**: Inspect before acting
   ```typescript
   const [deleteButton] = await page.observe("Find the delete button");
   if (deleteButton.description.includes("permanent")) {
     console.log("Warning: This action is permanent!");
     // Maybe ask for confirmation
   }
   await page.act(deleteButton);
   ```

5. **Debugging**: See what the AI found
   ```typescript
   const actions = await page.observe("Find the checkout button");
   console.log("AI found:", actions.map(a => ({
     description: a.description,
     selector: a.selector
   })));
   ```

### 3. `extract()` - Structured Data Extraction
```typescript
const { price } = await page.extract({
  instruction: "Get the product price",
  schema: z.object({ price: z.string() })
});
```
- **Purpose**: Pull structured data from the page
- **When to use**: When you need to read content, not interact
- **Returns**: Type-safe data matching your Zod schema

## Instruction Fidelity Guidelines

### ✅ Good Instructions

**Be action-oriented and specific about intent:**
- "Click the blue submit button"
- "Type 'hello@example.com' into the email field"
- "Select 'California' from the state dropdown"
- "Scroll to the testimonials section"

**Reference elements by their purpose or visible text:**
- "Click the 'Sign In' button" (uses button text)
- "Fill in the search box" (uses semantic role)
- "Click the shopping cart icon" (uses visual description)

### ❌ Avoid These Patterns

**Too vague:**
- "Do something with the form"
- "Handle the page"

**Technical implementation details:**
- "Click the div with class 'btn-primary'" (Stagehand uses semantic understanding, not CSS)
- "Find xpath://button[@id='submit']" (Don't use technical selectors)

**Multiple actions:**
- "Click submit and then wait and extract the result" (Break into separate calls)

### Instruction Flexibility

**Imperative (Recommended):**
```typescript
await page.act("Click the login button");
await page.act("Type 'user@example.com' in the email field");
```

**Declarative (Also Works):**
```typescript
await page.act("Submit the form"); // AI infers clicking submit button
await page.act("Search for 'stagehand'"); // AI infers typing + submitting
```

## The Complete Flow: Instruction → Playwright

### Step 1: Natural Language Instruction
```typescript
await page.act("Click the search box");
```

### Step 2: Accessibility Tree Analysis
Stagehand fetches the page's accessibility tree:
```
[0-147] button: Search... ⌘K
  [0-148] div
    [0-149] image
    [0-153] StaticText: Search...
```

### Step 3: AI Interpretation
The LLM analyzes the tree and returns:
```json
{
  "elementId": "0-147",
  "description": "Search button with keyboard shortcut",
  "method": "click",
  "arguments": []
}
```

### Step 4: XPath Resolution
Element ID `0-147` maps to XPath:
```
/html[1]/body[1]/div[1]/main[1]/div[1]/div[2]/div[1]/div[1]/div[2]/button[1]
```

### Step 5: Playwright Execution
```typescript
const locator = page.locator('xpath=/html[1]/body[1]/div[1]/main[1]/div[1]/div[2]/div[1]/div[1]/div[2]/button[1]');
await locator.click();
```

## Special Cases and Methods

### Filling Forms
```typescript
await page.act("Type 'Hello World' in the message box");
```
Translates to:
```typescript
await locator.fill("");
await locator.click();
for (const char of "Hello World") {
  await page.keyboard.type(char, { delay: 25-75ms });
}
```

### Scrolling
```typescript
await page.act("Scroll to 50%");
await page.act("Scroll to the footer");
```

### Keyboard Actions
```typescript
await page.act("Press Enter");
await page.act("Press Escape");
```

### Dropdowns
```typescript
// For <select> elements
await page.act("Select 'California' from the state dropdown");

// For custom dropdowns
await page.act("Choose 'Large' from the size menu");
```

## Best Practices

1. **Start general, get specific if needed:**
   ```typescript
   // Try this first
   await page.act("Click login");
   
   // If multiple login buttons exist, be more specific
   await page.act("Click the login button in the header");
   ```

2. **Use observe() for critical actions:**
   ```typescript
   const [deleteButton] = await page.observe("Find the delete account button");
   // Verify it's the right button before acting
   if (deleteButton.description.includes("permanently")) {
     await page.act(deleteButton);
   }
   ```

3. **Combine methods for complex flows:**
   ```typescript
   // Fill form
   await page.act("Enter 'John Doe' in the name field");
   await page.act("Enter 'john@example.com' in the email field");
   
   // Verify before submitting
   const [submitBtn] = await page.observe("Find the submit button");
   await page.act(submitBtn);
   
   // Extract confirmation
   const { orderId } = await page.extract({
     instruction: "Get the order confirmation number",
     schema: z.object({ orderId: z.string() })
   });
   ```

## Why This Works

1. **Semantic Understanding**: The accessibility tree provides role-based understanding (button, link, textbox) rather than implementation details
2. **Human-like Reasoning**: AI interprets instructions the way a human would - by understanding purpose and context
3. **Precise Execution**: XPath mapping ensures exact element targeting despite natural language ambiguity
4. **Resilient to Changes**: Even if CSS classes or IDs change, semantic roles and text content often remain stable

This approach makes browser automation as simple as giving instructions to a human user, while maintaining the precision and reliability needed for production use.