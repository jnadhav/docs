

### Question Paper: Senior React Developer Assessment for Bloomberg Terminal-Like UI

**Instructions**  
This assessment evaluates your ability to design and build a professional, data-intensive, React-based application similar to the Bloomberg Terminal UI. It consists of 20 tasks divided into three sets, to be completed in 90 minutes. Each task should take approximately 5-6 minutes. You may be asked to write code, explain approaches, or sketch designs. Focus on clean, performant, and scalable solutions. A coding environment will be provided for coding tasks. For discussion tasks, use a whiteboard or verbal explanations as needed.

---

### Set 1: Core React Skills for Bloomberg Terminal-Like UI (60 Minutes)

1. **Build a Real-Time Stock Ticker Component**  
   Create a `StockTicker` component that connects to a WebSocket endpoint (mocked as a function returning sample data) to display a stock’s symbol, price, and percentage change. Ensure the component updates in real-time and cleans up the WebSocket connection on unmount.

2. **Design a Resizable Panel Layout**  
   Explain how you would design a resizable, draggable panel system in React for displaying multiple data views (e.g., charts, tables). Provide a high-level component structure and state management approach.

3. **Implement a Data Table with Virtualization**  
   Create a `DataTable` component that renders a large dataset (10,000 rows) of stock data efficiently using virtualization. Use a library like `react-virtualized` or implement a basic version.

4. **Handle Real-Time API Data with Error Handling**  
   Create a `StockChart` component that fetches stock data from a REST API every 5 seconds and displays it. Include error handling for failed requests.

5. **Optimize Component Rendering with Memoization**  
   Given a `PriceDisplay` component that receives stock data as props, optimize it to avoid re-renders when the data hasn’t changed.

6. **Implement a Keyboard Shortcut Handler**  
   Write a `TableNavigator` component that allows users to navigate a table’s rows using arrow keys.

7. **Design a State Management Strategy**  
   Describe how you would manage state for a dashboard with multiple panels (charts, tables, forms) that share real-time data updates.

8. **Integrate a Charting Library**  
   Create a `StockChart` component that uses Chart.js to display a line chart of stock prices over time.

9. **Handle Concurrent API Requests**  
   Write a function that fetches stock data and news from two APIs concurrently and combines the results into a single state update.

10. **Discuss Testing Strategy for a Complex UI**  
    Describe how you would test a React application with multiple panels, real-time data, and complex interactions. Include tools and types of tests.

---

### Set 2: Security, Debugging, and Production-Grade Practices (30 Minutes)

11. **Implement Secure API Authentication**  
    Create a `SecureDataFetcher` component that fetches stock data from an API using OAuth 2.0 token authentication. Handle token expiration and refresh securely.

12. **Debug a Performance Issue in a React Component**  
    Given a `StockList` component that re-renders unnecessarily, identify the issue and propose a fix. The component displays a list of stocks and updates when new data arrives.

13. **Ensure Accessibility in a Form Component**  
    Modify a `SearchForm` component to be accessible, ensuring it meets WCAG 2.1 standards (e.g., keyboard navigation, screen reader support).

14. **Implement Error Boundary for Component Resilience**  
    Write an `ErrorBoundary` component to catch errors in a `StockChart` component and display a fallback UI.

15. **Secure WebSocket Communication**  
    Describe how to secure a WebSocket connection for real-time stock data and provide a code snippet to initialize it securely in a React component.

---

### Set 3: Complementary Tech Stacks for Modern React Applications (30 Minutes)

16. **TypeScript Integration for a Typed Component**  
    Convert a `StockCard` component to TypeScript, defining types for props and state, and handle a nullable data prop.

17. **Fetch Data with GraphQL**  
    Create a `StockDetails` component that uses Apollo Client to fetch stock data via a GraphQL query.

18. **Server-Side Rendering with Next.js**  
    Write a Next.js page that fetches stock data server-side and renders a dashboard with a `StockCard` component.

19. **Offload Heavy Computations to Web Workers**  
    Write a React component that offloads the calculation of a 50-day moving average for stock prices to a Web Worker.

20. **Integrate D3.js for Custom Data Visualization**  
    Write a `StockSparkline` component that uses D3.js to render a simple sparkline of stock prices.

---

**Notes for Candidates**  
- Allocate ~5-6 minutes per task.  
- For coding tasks, write partial or complete code as time allows and explain your approach.  
- For discussion tasks, provide clear, structured explanations, using whiteboarding if needed.  
- Focus on clean, performant, secure, and scalable solutions, reflecting the requirements of a Bloomberg Terminal-like UI.  

**End of Question Paper**

---

This question paper includes only the prompts for the 20 assignments, as requested, with no additional context, code samples, or evaluation criteria. You can copy this text into a document editor (e.g., Microsoft Word, Google Docs) to format and print or share with interviewers. If you need any modifications, such as formatting suggestions or additional instructions, please let me know!