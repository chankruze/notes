## Objective

Build a small React application that displays posts and allows users to view post details and comments.

Use the JSONPlaceholder API documentation:

[https://jsonplaceholder.typicode.com/guide/](https://jsonplaceholder.typicode.com/guide/)

---

## Technical Requirements

### Mandatory

- React
- React Router
- Axios (for all API requests)
- Functional Components
- Hooks (`useState`, `useEffect`)

> The examples below use `fetch` because they are copied from the API documentation. However, your implementation must use **Axios**.

### Nice to Have

- Loading states
- Error handling
- Responsive UI
- Clean code structure

---
# Task 1: Posts Listing Page

Create a page that lists all posts.

### API Reference

```js
fetch('https://jsonplaceholder.typicode.com/posts')
  .then((response) => response.json())
  .then((json) => console.log(json));
```

### Display Each Post As A Card

Show:

1. Post Title
2. Post Body
    - Maximum 3 lines
    - If content exceeds 3 lines, show ellipsis (`...`)
3. User Name
### Additional Requirements

- Show a loading state while data is being fetched.
- Show an error state if the request fails.
- Cards should be clickable.

---
# Task 2: Post Details Page

When a post card is clicked:

- Open the post details page in a new browser tab.

### Route Example

```txt
/posts/1
```

### API Reference

```js
fetch('https://jsonplaceholder.typicode.com/posts/1')
  .then((response) => response.json())
  .then((json) => console.log(json));
```

### Display

1. Post Title
2. Post Body
3. User Name

---
# Task 3: Comments Section

On the Post Details page, display all comments belonging to that post.

For each comment display:

1. Name
2. Email
3. Body

---
# Task 4: User Information

Display the author's name for every post and on the details page.

Use the API documentation to determine how user information can be retrieved.

---
# Evaluation Criteria

### React Fundamentals

- Component design
- Props usage
- State management
- Hooks usage

### API Integration

- Proper Axios usage
- Loading states
- Error handling

### Routing

- Route setup
- Dynamic routes

### Documentation Reading

- Ability to understand and use the API documentation
- Ability to discover required endpoints independently

### Code Quality

- Folder structure
- Reusable components
- Naming conventions
- Readability

---
# Submission

Share:

1. GitHub Repository URL
2. Setup Instructions (`README.md`)

---
# Bonus (Optional)

1. Search posts by title.
2. Add pagination.
3. Show comment count on each post card.
4. Cache API responses.
5. Use TanStack Query (React Query).
6. Implement a clean and scalable folder structure.