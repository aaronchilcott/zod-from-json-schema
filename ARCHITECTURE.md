# JSON Schema to Zod Converter Architecture

## Overview

This document describes the modular architecture for converting JSON Schema to Zod schemas. The architecture reflects JSON Schema's compositional nature, where each property adds independent constraints that combine to form the complete validation.

## Core Concepts

### Two-Phase Processing

The converter operates in two distinct phases based on a fundamental architectural principle:

**🔑 Key Principle: Refinement handlers should only be used when Zod doesn't support the operations natively.**

1. **Primitive Phase**: Handlers that use Zod's built-in constraint methods (`.min()`, `.max()`, `.regex()`, etc.)
2. **Refinement Phase**: Handlers that add custom validation logic through Zod's `.refine()` method for operations Zod cannot express natively

### Type Schemas

During the first phase, we maintain a `TypeSchemas` object that tracks the state of each possible type:

```typescript
interface TypeSchemas {
  string?: z.ZodTypeAny | false;
  number?: z.ZodTypeAny | false;  // integers are numbers with .int() constraint
  boolean?: z.ZodTypeAny | false;
  null?: z.ZodNull | false;
  array?: z.ZodArray<any> | false;
  tuple?: z.ZodTuple<any> | false;
  object?: z.ZodObject<any> | false;
}
```

Each type can be in one of three states:
- `undefined`: Type is still allowed (no constraints have excluded it)
- `false`: Type is explicitly disallowed
- `z.Zod*`: Type with accumulated constraints (including literals, enums, and unions)

## Architecture Components

### 1. Primitive Handlers

These handlers operate during the first phase and modify the `TypeSchemas`:

```typescript
interface PrimitiveHandler {
  apply(types: TypeSchemas, schema: JSONSchema): void;
}
```

#### Implemented Primitive Handlers:
- **TypeHandler**: Sets types to `false` if not in the `type` array
- **ConstHandler**: Handles const values by creating literals
- **EnumHandler**: Handles enum validation with appropriate Zod types
- **ImplicitStringHandler**: Enables string type when string constraints are present without explicit type
- **MinLengthHandler**: Applies `.min()` to string (Zod native support)
- **MaxLengthHandler**: Applies `.max()` to string (Zod native support)
- **PatternHandler**: Applies `.regex()` to string (Zod native support)
- **MinimumHandler**: Applies `.min()` to number (Zod native support)
- **MaximumHandler**: Applies `.max()` to number (Zod native support)
- **ExclusiveMinimumHandler**: Applies `.gt()` to number (Zod native support)
- **ExclusiveMaximumHandler**: Applies `.lt()` to number (Zod native support)
- **MultipleOfHandler**: Applies `.multipleOf()` to number (Zod native support)
- **MinItemsHandler**: Applies `.min()` to array (Zod native support)
- **MaxItemsHandler**: Applies `.max()` to array (Zod native support)
- **ItemsHandler**: Configures array element validation if arrays still allowed
- **TupleHandler**: Detects tuple arrays and marks them as tuple type
- **PropertiesHandler**: Creates initial object schema with known properties

### 2. Refinement Handlers

These handlers operate during the second phase on the combined schema:

```typescript
interface RefinementHandler {
  apply(zodSchema: z.ZodTypeAny, schema: JSONSchema): z.ZodTypeAny;
}
```

#### Implemented Refinement Handlers:

**✅ Legitimate Refinement Handlers (Zod doesn't support natively):**
- **UniqueItemsHandler**: Custom validation for array uniqueness (Zod has no native unique constraint)
- **NotHandler**: Complex logical negation validation (Zod has no native `.not()`)
- **AllOfHandler**: Complex schema intersection logic
- **AnyOfHandler**: Complex anyOf validation logic  
- **OneOfHandler**: Complex oneOf validation (exactly one must match)
- **EnumComplexHandler**: Complex object/array equality in enums
- **ConstComplexHandler**: Complex object/array equality for const values
- **MetadataHandler**: Description and title annotations

**⚠️ Edge Case Handlers (legitimate but specific):**
- **ProtoRequiredHandler**: Special handler for `__proto__` security protection
- **EmptyEnumHandler**: Handles empty enum arrays (always invalid)
- **EnumNullHandler**: Handles null in enum when type doesn't include null
- **PrefixItemsHandler**: Handles Draft 2020-12 prefixItems validation
- **ObjectPropertiesHandler**: Object validation for union types (primitive work moved to PropertiesHandler)

## Processing Flow

### Phase 1: Type-Specific Constraints

1. Initialize empty `TypeSchemas` object
2. Run all primitive handlers in sequence
3. Each handler:
   - Checks if it has relevant constraints in the schema
   - For each type it affects, checks if that type is still allowed (`!== false`)
   - If allowed and constraint applies, either:
     - Creates initial type schema if `undefined`
     - Adds constraints to existing type schema

### Phase 2: Build Union and Apply Refinements

1. Convert remaining `undefined` types to their most permissive schemas:
   - `string` → `z.string()`
   - `number` → `z.number()`
   - `array` → `z.array(z.any())`
   - `tuple` → handled by TupleItemsHandler
   - `object` → `z.object({}).passthrough()`
   - etc.

2. Filter out `false` types and create union of allowed types:
   - 0 types → `z.never()`
   - 1 type → that type's schema
   - 2+ types → `z.union([...])`

3. Run all refinement handlers on the resulting schema

## Example: Processing a Complex Schema

Given this JSON Schema:
```json
{
  "type": ["string", "number"],
  "minimum": 5,
  "minLength": 3,
  "pattern": "^[A-Z]",
  "uniqueItems": true
}
```

**Phase 1 (Primitive Handlers):**
1. TypeHandler: marks `boolean`, `null`, `array`, `object` as `false`
2. MinimumHandler: sets `number` to `z.number().min(5)`
3. MinLengthHandler: sets `string` to `z.string().min(3)`
4. PatternHandler: updates `string` to `z.string().min(3).regex(/^[A-Z]/)`

**Result after Phase 1:**
```typescript
{
  string: z.string().min(3).regex(/^[A-Z]/),
  number: z.number().min(5),
  boolean: false,
  null: false,
  array: false,
  tuple: undefined,
  object: false
}
```

**Phase 2:**
1. Create union: `z.union([z.string().min(3).regex(/^[A-Z]/), z.number().min(5)])`
2. UniqueItemsHandler: Adds refinement (only validates for arrays, but none allowed here)

## Implementation Status

### Test Results
- **Total tests**: 1355 (999 active, 356 skipped)
- **Passing**: 999 tests
- **Failing**: 0 tests
- **Skipped**: 356 tests (JSON Schema features not supported by Zod)

### Known Limitations
1. **`__proto__` property validation**: Zod's `passthrough()` strips this property for security. Solved with ProtoRequiredHandler using `z.any()` when `__proto__` is required.
2. **Unicode grapheme counting**: JavaScript uses UTF-16 code units instead of grapheme clusters. Test added to skip list as platform limitation.
3. **Complex schema combinations**: Some edge cases with deeply nested `allOf`, `anyOf`, `oneOf` combinations may not perfectly match JSON Schema semantics.

## Benefits

1. **Modularity**: Each JSON Schema keyword is handled by a dedicated handler
2. **Composability**: Handlers don't need to know about each other
3. **Type Safety**: Type-specific constraints are only applied to appropriate types
4. **Extensibility**: New keywords can be supported by adding new handlers
5. **Maintainability**: Clear separation between constraint types
6. **Correctness**: Reflects JSON Schema's additive constraint model
7. **Testability**: Each handler can be tested independently
8. **Performance**: Native Zod operations are faster than custom refinements
9. **Better Type Inference**: Primitive handlers create proper Zod types with built-in validation
10. **Architectural Clarity**: Clear distinction between schema construction vs. custom validation

## Implementation Guidelines

### Adding a New Primitive Handler

**Use primitive handlers when Zod has native support for the constraint (e.g., `.min()`, `.max()`, `.regex()`).**

1. Determine which type(s) the constraint affects
2. Create handler that checks if those types are still allowed
3. Apply constraints using Zod's built-in methods (prefer native over custom logic)
4. Add type guards when working with `z.ZodTypeAny` to ensure type safety
5. Consider if the constraint should enable a type implicitly (like `ImplicitStringHandler`)

Example:
```typescript
export class MyConstraintHandler implements PrimitiveHandler {
    apply(types: TypeSchemas, schema: JSONSchema.BaseSchema): void {
        const mySchema = schema as JSONSchema.MySchema;
        if (mySchema.myConstraint === undefined) return;
        
        if (types.string !== false) {
            const currentString = types.string || z.string();
            if (currentString instanceof z.ZodString) {
                types.string = currentString.myMethod(mySchema.myConstraint);
            }
        }
    }
}
```

### Adding a New Refinement Handler

**Only use refinement handlers when Zod doesn't support the operation natively.**

1. Use for constraints that:
   - **Cannot be expressed with Zod's built-in constraints** (e.g., uniqueItems, complex object equality)
   - Apply complex logical operations (e.g., not, anyOf, oneOf)
   - Require custom validation across multiple types
   - Handle edge cases or security concerns
2. Handler receives the complete schema after type union
3. Return schema with added `.refine()` validation
4. **Avoid using refinements for operations Zod supports natively** (e.g., string length, number ranges)

Example:
```typescript
export class MyRefinementHandler implements RefinementHandler {
    apply(zodSchema: z.ZodTypeAny, schema: JSONSchema.BaseSchema): z.ZodTypeAny {
        if (!schema.myConstraint) return zodSchema;
        
        return zodSchema.refine(
            (value: any) => {
                // Custom validation logic
                return validateMyConstraint(value, schema.myConstraint);
            },
            { message: "Value does not satisfy myConstraint" }
        );
    }
}
```

### Handler Order

- **Primitive handlers**: Order matters for some handlers:
  - ConstHandler and EnumHandler should run before TypeHandler
  - ImplicitStringHandler should run before other string handlers
  - TupleHandler should run before ItemsHandler
  - Others can run in any order (they're independent)
  
- **Refinement handlers**: Should be ordered by complexity/dependencies:
  - Special cases first (ProtoRequiredHandler, EmptyEnumHandler)
  - Logical combinations (AllOf, AnyOf, OneOf)
  - Type-specific refinements (TupleItems, ArrayItems, ObjectProperties)
  - General refinements (Not, UniqueItems)
  - Metadata handlers last

## Future Enhancements

1. **Additional JSON Schema Keywords**: Support for more keywords like `dependencies`, `if/then/else`, `contentMediaType`, etc.
2. **Performance Optimization**: Cache converted schemas for repeated conversions
3. **Better Error Messages**: Provide more descriptive validation error messages
4. **Schema Version Support**: Handle different JSON Schema draft versions
5. **Bidirectional Conversion**: Improve Zod to JSON Schema conversion fidelity

## Architectural Evolution

### Key Insight: Native vs. Custom Validation

During development, we discovered that **many operations initially implemented as refinement handlers should actually be primitive handlers** because Zod supports them natively. This led to a major architectural insight:

**❌ Anti-pattern: Using refinements for Zod-native operations**
```typescript
// WRONG: Using refinement for string length (Zod supports .min() natively)
return zodSchema.refine(
    (value: any) => typeof value !== "string" || value.length >= minLength,
    { message: "String too short" }
);
```

**✅ Correct pattern: Using primitive handlers for Zod-native operations**
```typescript
// CORRECT: Using Zod's native .min() method
if (types.string !== false) {
    types.string = (types.string || z.string()).min(minLength);
}
```

### Migration Examples

1. **String Constraints**: Moved from `StringConstraintsHandler` (refinement) to `ImplicitStringHandler` + existing primitive handlers
2. **Array Items**: Moved from `ArrayItemsHandler` (refinement) to enhanced `ItemsHandler` (primitive) + `PrefixItemsHandler` (refinement for edge cases)
3. **Tuple Handling**: Moved from `TupleItemsHandler` (refinement) to `TupleHandler` (primitive)

### Benefits of This Evolution

- **Performance**: Native Zod methods are faster than custom refinements
- **Type Safety**: Better TypeScript inference with proper Zod types
- **Maintainability**: Less custom validation code to maintain
- **Coverage**: Eliminated unreachable code paths in refinement handlers

## Conclusion

The modular two-phase architecture successfully addresses the need for a clean, extensible design where each JSON Schema property is handled by independent modules. The key insight about **preferring native Zod operations over custom refinements** has significantly improved the architecture's performance, maintainability, and correctness.

This approach makes the codebase more maintainable, testable, and easier to extend with new JSON Schema features while leveraging Zod's full capabilities.