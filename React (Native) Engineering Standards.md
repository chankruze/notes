## 1. Engineering Principles

### Core Values

1. Readability over cleverness.
2. Explicit over implicit.
3. Composition over inheritance.
4. Predictability over shortcuts.
5. Scalability over convenience.
6. Consistency over personal preference.
### Golden Rule

Any engineer should be able to open a file and understand:
- What it does
- Why it exists
- Where business logic lives
- Where data comes from
- How to modify it safely

within 2-3 minutes.
## 2. Feature Architecture

Every feature is self-contained.

```text
src/
├── screens/
│
├── shared/
│   ├── components/
│   ├── hooks/
│   ├── utils/
│   ├── services/
│   ├── constants/
│   ├── types/
│   └── providers/
│
├── navigation/
├── theme/
├── assets/
└── config/
```

Feature Example:

```text
packages/
├── api/
├── components/
├── hooks/
├── services/
├── constants/
├── utils/
├── types/
├── sections/
├── layouts/
├── screen.tsx
└── layout.tsx
```
### Dependency Direction

Allowed:

```text
screen
  ↓
sections
  ↓
components
  ↓
hooks
  ↓
services
  ↓
api
```

Forbidden:

```text
components -> screen
services -> components
hooks -> UI
api -> hooks
```

Dependencies must flow downward only.
# 3. File Size Limits

### Components

```text
Preferred: < 150 lines
Maximum: 250 lines
```
### Hooks

```text
Preferred: < 200 lines
Maximum: 300 lines
```
### Screen Files

```text
Preferred: < 80 lines
Maximum: 120 lines
```
### Functions

```text
Preferred: < 30 lines
Maximum: 50 lines
```

If exceeded, extract.
# 4. Naming Standards

## Files

Always kebab-case.

```text
package-card.tsx
package-details-section.tsx
use-package-list.ts
package-service.ts
package-constants.ts
```

Never:

```text
PackageCard.tsx
packageCard.tsx
```
## Components

Always PascalCase.

```tsx
export const PackageCard = () => {};
```
## Hooks

```tsx
use-package-list.ts
use-package-filters.ts
use-user-profile.ts
```
## Constants

```tsx
MAX_RETRY_ATTEMPTS
DEFAULT_PAGE_SIZE
BOTTOM_SHEET_HEIGHT
```
## Types

```tsx
PackageSummary
PackageFilters
PackageResponse
```

Interfaces should be suffixed only when necessary:

```tsx
PackageCardProps
```
# 5. React Component Standards

### Component Responsibilities

A component should do ONE thing.

Bad:

```tsx
PackageCard
- fetches data
- transforms data
- handles pagination
- renders UI
```

Good:

```tsx
PackageCard
- renders package
```
### Component Structure

Order:

```tsx
imports
types
constants
component
styles
export
```

Example:

```tsx
type PackageCardProps = {
  package: PackageSummary;
  onPress: (id: string) => void;
};

export const PackageCard = React.memo(
  ({ package, onPress }: PackageCardProps) => {
    return (
      <Pressable onPress={() => onPress(package.id)}>
        ...
      </Pressable>
    );
  }
);
```
# 6. Hook Standards

Hooks own business logic.

```tsx
use-package-list.ts
```

Contains:
- API calls
- filtering
- pagination
- transformations
- derived state

Not allowed:

```tsx
return (
  <View />
);
```

No JSX in hooks.
### Hook Return Pattern

Prefer:

```tsx
return {
  packages,
  isLoading,
  error,
  loadMore,
  refresh,
};
```

Avoid:

```tsx
return [
  packages,
  isLoading,
  error,
];
```

Object returns are self-documenting.

# 7. API Layer Standards

## Never Call APIs Directly Inside Components

Bad:

```tsx
useEffect(() => {
  axios.get(...);
}, []);
```

Good:

```tsx
const { data } = usePackageList();
```

## API Layer

```text
api/
├── package-api.ts
├── user-api.ts
```

Example:

```tsx
export const getPackages = async (
  params: PackageQueryParams
): Promise<PackageResponse> => {
  return apiClient.get(...);
};
```

API files should contain:
- request construction
- response typing

Nothing else.
# 8. State Management Rules

## Local State

Use:

```tsx
useState
```

For:
- modal open state
- selected tab
- form field values

## Derived State

Use:

```tsx
useMemo
```

Never store derived values.

Bad:

```tsx
const [filteredUsers, setFilteredUsers] = useState([]);
```

Good:

```tsx
const filteredUsers = useMemo(...);
```

## Global State

Use Zustand only for:
- auth
- user session
- app configuration
- feature flags

Never use global state for screen-specific data.
# 9. React Query Standards

All server state should use [Tanstack Query](https://tanstack.com/query/latest)
### Query Keys

```tsx
query-keys.ts
```

```tsx
export const QUERY_KEYS = {
  PACKAGES: ["packages"],
  USER_PROFILE: ["user-profile"],
};
```

Never hardcode query keys.
### Mutations

Always invalidate relevant queries.

```tsx
onSuccess: () => {
  queryClient.invalidateQueries({
    queryKey: QUERY_KEYS.PACKAGES,
  });
};
```

# 10. Styling Standards (React Native)

## Design Tokens Only

Allowed:

```tsx
theme.colors.primary
theme.spacing.md
theme.radius.lg
```

Not allowed:

```tsx
"#FF0000"
16
24
```

## Style Location (React Native)

Small component:

```tsx
component.tsx
component.styles.ts
```

Large feature:

```text
styles/
├── package-card.styles.ts
```

# 11. List Rendering Standards (React Native)

### FlashList Only

Use:

```tsx
@shopify/flash-list
```

Avoid FlatList unless justified.
### Item Extraction

Bad:

```tsx
renderItem={({ item }) => (
  <PackageCard item={item} />
)}
```

Good:

```tsx
const renderItem = useCallback(
  ({ item }: ListRenderItemInfo<Package>) => (
    <PackageCard item={item} />
  ),
  []
);
```

### Keys

Never:

```tsx
index.toString()
```

Always:

```tsx
item.id
```

# 12. Error Handling Standards

Never silently swallow errors.

Bad:

```tsx
catch (error) {}
```

Good:

```tsx
catch (error) {
  logger.error(error);
}
```

### User-facing Errors

Always provide the following states:

```tsx
loading
error
empty
success
```

# 13. Logging Standards

Use centralized logger.

```tsx
logger.info()
logger.warn()
logger.error()
```

Never:

```tsx
console.log()
console.error()
```

in production code.

# 14. Testing Standards

## Unit Tests

Required for:
- utilities
- hooks
- business logic

```text
calculate-total-price.test.ts
```
## Component Tests

Test:
- user interaction
- rendering states

Not implementation details.

## Coverage

Minimum:

```text
80%
```

For critical business logic:

```text
90%+
```
# 15. Accessibility Standards

Every interactive element must have:

```tsx
accessibilityLabel
accessibilityRole
```

Example:

```tsx
<Button
  accessibilityLabel="Submit package"
  accessibilityRole="button"
/>
```
# 16. Security Standards

Never store:

```text
tokens
passwords
api keys
```

in source code.

Use:

```text
env
secure storage (React Native)
```

Never trust client data.

Validate:
- API responses
- route params
- deep links
- storage values

# 17. Git Standards

### Branch Naming

```text
feature/COND-6075-pdf-generation-for-cheques
fix/COND-6074-pdf-generation-for-cheques
refactor/COND-6077-package-hooks
```
### Commit Format

```text
COND-6071: add package search
COND-6072: resolve package pagination bug
COND-6073: reafactor package filters hook
COND-6074: add package service tests
```
# 18. Pull Request Standards

PR must:
- Have a clear description.
- Include screenshots/video for UI changes.
- Explain architectural decisions.
- Be under 500 changed lines where possible.

Checklist:

```text
□ Types added
□ Tests added
□ No dead code
□ No console logs
□ No any
□ Uses theme tokens
□ Accessibility verified
□ Query invalidation verified
```
# 19. Forbidden Patterns

Never:

```tsx
any
```

Never:

```tsx
export * from ...
```

Never:

```tsx
../../../
```

Never:

```tsx
large screen files
```

Never:

```tsx
business logic inside JSX
```

Never:

```tsx
API calls inside components
```

Never:

```tsx
magic strings
```

Never:

```tsx
console.log()
```

Never:

```tsx
index as React key
```

Never:

```tsx
inline styles in screen.tsx
```

Never:

```tsx
50-line useEffect()
```
# 20. Definition of Done (DoD)

A feature is complete only when:

- [ ] Architecture follows feature structure.
- [ ] Screen is orchestration-only.
- [ ] Business logic extracted to hooks.
- [ ] API logic extracted to api/services.
- [ ] No barrel exports.
- [ ] All imports use `@/`.
- [ ] No `any`.
- [ ] No dead code.
- [ ] No magic values.
- [ ] Components are memoized where appropriate.
- [ ] FlashList follows v2 standards.
- [ ] Error, Loading, Empty states implemented.
- [ ] Accessibility added.
- [ ] Tests added.
- [ ] PR checklist passed.
- [ ] Code review approved.

This gives you a standards document close to what larger React Native teams use internally, while remaining practical for a startup/team of 3–20 engineers.