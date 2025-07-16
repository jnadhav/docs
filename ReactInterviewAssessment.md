---

### 20 Mini Assessments for Senior React Developer: Bloomberg Terminal-Like UI

**Introduction**  
To assess a senior React developer's ability to design and build a professional, data-intensive, end-to-end application similar to the Bloomberg Terminal UI using the React tech stack, we have designed 20 mini assignments/questions. These are divided into three sets:  
1. **Set 1 (10 tasks, 60 minutes)**: Core React skills, real-time data handling, and UI design.  
2. **Set 2 (5 tasks, 30 minutes)**: Security, debugging, and production-grade practices.  
3. **Set 3 (5 tasks, 30 minutes)**: Complementary tech stacks (TypeScript, GraphQL, Next.js, Web Workers, D3.js).  

Each task is designed to take ~5-6 minutes, totaling a 90-minute in-person session. The assessments evaluate skills critical for building a Bloomberg Terminal-like application, including React expertise, real-time data movement, API integration, performance optimization, security, and complementary technologies. Below are the detailed tasks, code samples, evaluation criteria, and context.

**Clarifications and Assumptions**  
- **Scope**: Greenfield React-based front-end with heavy data interaction (APIs, WebSockets, events) and a data-dense UI.  
- **Tech Stack**: React (with Hooks), TypeScript, Redux/Context API, WebSocket, REST APIs, modern CSS (e.g., CSS-in-JS, Tailwind, Styled-Components).  
- **Assessment Format**: Each task includes coding, explanation, or discussion within ~5-6 minutes.  
- **Bloomberg Terminal UI Characteristics**:  
  - **Data Density**: Displays real-time financial data (e.g., stock prices, news, charts).  
  - **Real-Time Updates**: Uses WebSockets or polling for live data feeds.  
  - **Modular Layout**: Multiple resizable, draggable panels with charts, tables, and forms.  
  - **Interactivity**: Keyboard shortcuts, dynamic forms, responsive design.  
  - **Performance**: Handles large datasets without lag, using virtualization or memoization.  
- **Evaluation Criteria**: Clean, performant code; real-time data handling; scalable component architecture; user experience; security; and complementary tech proficiency.

**Skills Required**  
1. React Expertise: Component-based architecture, Hooks, Context API, Redux.  
2. TypeScript: Type-safe code for scalability.  
3. Real-Time Data: WebSocket integration for live data.  
4. API Integration: REST, GraphQL, error handling, caching.  
5. UI/UX Design: Modular, responsive layouts with data visualization.  
6. Performance Optimization: Virtualization, memoization, lazy loading.  
7. Event Handling: User and server-sent events.  
8. Scalability: Reusable code, design systems.  
9. Data Visualization: Libraries like D3.js, Chart.js, HighCharts.  
10. Testing: Jest, React Testing Library for unit/integration tests.  
11. Security: Secure API/WebSocket communication, authentication.  
12. Debugging: Identifying and resolving performance issues.  
13. Production Readiness: Error boundaries, accessibility, monitoring.

---

### Set 1: Core React Skills for Bloomberg Terminal-Like UI (60 Minutes)

#### 1. Build a Real-Time Stock Ticker Component
**Task**: Write a React component that displays a stock ticker with real-time price updates using WebSocket. The component should show the stock symbol, price, and percentage change, updating every second.  
**Time**: 6 minutes  
**Skills Tested**: React Hooks, WebSocket integration, state management, real-time UI updates.  
**Prompt**:  
"Create a `StockTicker` component that connects to a WebSocket endpoint (mocked as a function returning sample data) to display a stock’s symbol, price, and percentage change. Ensure the component updates in real-time and cleans up the WebSocket connection on unmount."

**Sample Code**:
```jsx
import React, { useState, useEffect } from 'react';

interface StockData {
  symbol: string;
  price: number;
  change: number;
}

const StockTicker: React.FC<{ symbol: string }> = ({ symbol }) => {
  const [stockData, setStockData] = useState<StockData | null>(null);

  useEffect(() => {
    // Mock WebSocket connection
    const ws = new WebSocket('wss://mock-stock-api.com');
    ws.onmessage = (event) => {
      const data: StockData = JSON.parse(event.data);
      if (data.symbol === symbol) {
        setStockData(data);
      }
    };

    // Mock data for testing
    const mockData = () => ({
      symbol,
      price: Math.random() * 100 + 100,
      change: Math.random() * 5 - 2.5,
    });

    const interval = setInterval(() => {
      setStockData(mockData());
    }, 1000);

    return () => {
      ws.close();
      clearInterval(interval);
    };
  }, [symbol]);

  if (!stockData) return <div>Loading...</div>;

  return (
    <div style={{ padding: '10px', border: '1px solid #ccc' }}>
      <h3>{stockData.symbol}</h3>
      <p>Price: ${stockData.price.toFixed(2)}</p>
      <p style={{ color: stockData.change >= 0 ? 'green' : 'red' }}>
        Change: {stockData.change.toFixed(2)}%
      </p>
    </div>
  );
};

export default StockTicker;
```

**Expected Answer/Discussion**:  
- **Code Explanation**: Use `useEffect` to manage WebSocket lifecycle, ensuring cleanup on unmount to prevent memory leaks. Update UI with state changes. TypeScript ensures type safety.  
- **Evaluation**: Check for proper WebSocket handling, cleanup, and real-time UI updates. Bonus for TypeScript usage and error handling (e.g., WebSocket connection failure).  
- **Bloomberg Relevance**: Real-time stock tickers are core to the Terminal’s UI, requiring efficient data updates and minimal re-renders.  
**Evaluation Criteria**:  
- Correct use of `useEffect` and state management (2 points).  
- WebSocket lifecycle handling and cleanup (2 points).  
- UI rendering with dynamic styles (1 point).

#### 2. Design a Resizable Panel Layout
**Task**: Describe and sketch a component architecture for a resizable, draggable panel layout (like Bloomberg Terminal’s multi-window interface) using React.  
**Time**: 5 minutes  
**Skills Tested**: Component architecture, UI design, state management.  
**Prompt**:  
"Explain how you would design a resizable, draggable panel system in React for displaying multiple data views (e.g., charts, tables). Provide a high-level component structure and state management approach."

**Sample Answer**:  
- **Architecture**:  
  - `Dashboard`: Root component managing layout state (panel positions, sizes).  
  - `Panel`: Individual resizable/draggable component containing content (e.g., chart, table).  
  - `ResizableContainer`: Wrapper using a library like `react-resizable` for resizing.  
  - `DragContext`: Context API to manage drag state across panels.  
- **State Management**: Use Redux or Context API to store panel configurations (e.g., `{ id, x, y, width, height }`). Update state on drag/resize events.  
- **Implementation**:  
  - Use `react-draggable` for drag functionality and `react-resizable` for resizing.  
  - Store panel state in a reducer or Context, updating on drag/resize end.  
  - Render panels dynamically based on state, ensuring modularity.  
- **Example Pseudo-Code**:
```jsx
const Dashboard: React.FC = () => {
  const [panels, setPanels] = useState([
    { id: '1', x: 0, y: 0, width: 400, height: 300, content: <Chart /> },
  ]);

  return (
    <div style={{ position: 'relative', height: '100vh' }}>
      {panels.map((panel) => (
        <ResizableContainer key={panel.id} panel={panel} onResize={updatePanel} />
      ))}
    </div>
  );
};
```

**Evaluation**:  
- **Key Points**: Propose a modular architecture, mention libraries for drag/resize, explain state persistence. Bonus for discussing performance (e.g., debouncing resize events).  
- **Bloomberg Relevance**: The Terminal’s UI relies on flexible, user-configurable layouts for multitasking.  
**Evaluation Criteria**:  
- Clear component hierarchy (2 points).  
- State management strategy (2 points).  
- Mention of drag/resize libraries or techniques (1 point).

#### 3. Implement a Data Table with Virtualization
**Task**: Write a React component for a virtualized data table displaying 10,000 rows of financial data (e.g., stock prices).  
**Time**: 6 minutes  
**Skills Tested**: Performance optimization, virtualization, React rendering.  
**Prompt**:  
"Create a `DataTable` component that renders a large dataset (10,000 rows) of stock data efficiently using virtualization. Use a library like `react-virtualized` or implement a basic version."

**Sample Code**:
```jsx
import React from 'react';
import { List } from 'react-virtualized';

interface Stock {
  id: string;
  symbol: string;
  price: number;
}

const DataTable: React.FC<{ data: Stock[] }> = ({ data }) => {
  const rowHeight = 30;
  const height = 400;
  const width = 600;

  const rowRenderer = ({ index, style }: { index: number; style: React.CSSProperties }) => {
    const stock = data[index];
    return (
      <div style={{ ...style, display: 'flex' }} key={stock.id}>
        <div style={{ flex: 1 }}>{stock.symbol}</div>
        <div style={{ flex: 1 }}>${stock.price.toFixed(2)}</div>
      </div>
    );
  };

  return (
    <List
      width={width}
      height={height}
      rowCount={data.length}
      rowHeight={rowHeight}
      rowRenderer={rowRenderer}
    />
  );
};

export default DataTable;
```

**Evaluation**:  
- **Key Points**: Use `react-virtualized` to render only visible rows, reducing DOM overhead. `rowRenderer` maps data efficiently.  
- **Evaluation**: Check virtualization, handling of large datasets, clean structure. Bonus for TypeScript or custom virtualization logic.  
- **Bloomberg Relevance**: The Terminal displays large datasets with smooth scrolling.  
**Evaluation Criteria**:  
- Virtualization implementation (2 points).  
- Correct row rendering (2 points).  
- Performance considerations (1 point).

#### 4. Handle Real-Time API Data with Error Handling
**Task**: Write a React component that fetches real-time stock data from a REST API and handles errors gracefully.  
**Time**: 6 minutes  
**Skills Tested**: API integration, error handling, React Hooks.  
**Prompt**:  
"Create a `StockChart` component that fetches stock data from a REST API every 5 seconds and displays it. Include error handling for failed requests."

**Sample Code**:
```jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

interface StockData {
  symbol: string;
  price: number;
}

const StockChart: React.FC<{ symbol: string }> = ({ symbol }) => {
  const [data, setData] = useState<StockData | null>(null);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    const fetchData = async () => {
      try {
        const response = await axios.get(`https://api.example.com/stocks/${symbol}`);
        setData(response.data);
        setError(null);
      } catch (err) {
        setError('Failed to fetch data');
      }
    };

    fetchData();
    const interval = setInterval(fetchData, 5000);

    return () => clearInterval(interval);
  }, [symbol]);

  if (error) return <div>Error: {error}</div>;
  if (!data) return <div>Loading...</div>;

  return (
    <div>
      <h3>{data.symbol}</h3>
      <p>Price: ${data.price.toFixed(2)}</p>
    </div>
  );
};

export default StockChart;
```

**Evaluation**:  
- **Key Points**: Use `useEffect` for polling, handle errors with try-catch, clean up intervals. Bonus for retry logic or caching.  
- **Bloomberg Relevance**: Robust API integration for real-time market data.  
**Evaluation Criteria**:  
- API fetching and polling (2 points).  
- Error handling (2 points).  
- Cleanup logic (1 point).

#### 5. Optimize Component Rendering with Memoization
**Task**: Optimize a React component to prevent unnecessary re-renders using `React.memo` and `useMemo`.  
**Time**: 5 minutes  
**Skills Tested**: Performance optimization, React rendering.  
**Prompt**:  
"Given a `PriceDisplay` component that receives stock data as props, optimize it to avoid re-renders when the data hasn’t changed."

**Sample Code**:
```jsx
import React, { memo, useMemo } from 'react';

interface PriceDisplayProps {
  symbol: string;
  price: number;
}

const PriceDisplay: React.FC<PriceDisplayProps> = ({ symbol, price }) => {
  const formattedPrice = useMemo(() => price.toFixed(2), [price]);

  return (
    <div>
      <h3>{symbol}</h3>
      <p>Price: ${formattedPrice}</p>
    </div>
  );
};

export default memo(PriceDisplay);
```

**Evaluation**:  
- **Key Points**: Use `React.memo` for the component, `useMemo` for computed values, explain re-render prevention.  
- **Bloomberg Relevance**: Performance is critical under heavy data updates.  
**Evaluation Criteria**:  
- Use of `React.memo` (2 points).  
- Use of `useMemo` (2 points).  
- Explanation of optimization (1 point).

#### 6. Implement a Keyboard Shortcut Handler
**Task**: Add keyboard shortcut support to a React component for navigating a data table (e.g., arrow keys to move selection).  
**Time**: 5 minutes  
**Skills Tested**: Event handling, UX design.  
**Prompt**:  
"Write a `TableNavigator` component that allows users to navigate a table’s rows using arrow keys."

**Sample Code**:
```jsx
import React, { useState, useEffect } from 'react';

const TableNavigator: React.FC<{ rows: string[] }> = ({ rows }) => {
  const [selectedIndex, setSelectedIndex] = useState(0);

  useEffect(() => {
    const handleKeyDown = (e: KeyboardEvent) => {
      if (e.key === 'ArrowDown' && selectedIndex < rows.length - 1) {
        setSelectedIndex((prev) => prev + 1);
      } else if (e.key === 'ArrowUp' && selectedIndex > 0) {
        setSelectedIndex((prev) => prev - 1);
      }
    };

    window.addEventListener('keydown', handleKeyDown);
    return () => window.removeEventListener('keydown', handleKeyDown);
  }, [selectedIndex, rows.length]);

  return (
    <div>
      {rows.map((row, index) => (
        <div
          key={index}
          style={{
            background: index === selectedIndex ? 'lightblue' : 'white',
            padding: '5px',
          }}
        >
          {row}
        </div>
      ))}
    </div>
  );
};

export default TableNavigator;
```

**Evaluation**:  
- **Key Points**: Handle keyboard events, manage selection state, clean up listeners. Bonus for accessibility (e.g., focus management).  
- **Bloomberg Relevance**: Keyboard shortcuts enable rapid navigation.  
**Evaluation Criteria**:  
- Keyboard event handling (2 points).  
- State management for selection (2 points).  
- Event cleanup (1 point).

#### 7. Design a State Management Strategy
**Task**: Propose a state management solution for a Bloomberg Terminal-like app with multiple panels and real-time data.  
**Time**: 5 minutes  
**Skills Tested**: State management, system design.  
**Prompt**:  
"Describe how you would manage state for a dashboard with multiple panels (charts, tables, forms) that share real-time data updates."

**Sample Answer**:  
- **Solution**: Use Redux for centralized state management to handle shared data (e.g., stock prices, user settings). Panels subscribe to state slices.  
- **Structure**:  
  - **Store**: Single Redux store with reducers for `stocks`, `layout`, `user`.  
  - **Slices**:  
    - `stocks`: Real-time stock data via WebSocket middleware.  
    - `layout`: Panel positions and sizes.  
    - `user`: User preferences (e.g., theme, shortcuts).  
  - **Middleware**: Custom middleware for WebSocket to dispatch actions.  
- **Alternative**: Context API for smaller apps, but Redux for scalability.  
- **Example**:
```javascript
const stockSlice = createSlice({
  name: 'stocks',
  initialState: {},
  reducers: {
    updateStock: (state, action) => {
      state[action.payload.symbol] = action.payload;
    },
  },
});
```

**Evaluation**:  
- **Key Points**: Justify Redux/Context API, explain data flow, address real-time updates. Bonus for middleware or TypeScript.  
- **Bloomberg Relevance**: Centralized state ensures consistent data across panels.  
**Evaluation Criteria**:  
- Clear state management strategy (2 points).  
- Handling of real-time data (2 points).  
- Scalability considerations (1 point).

#### 8. Integrate a Charting Library
**Task**: Write a React component that integrates a charting library (e.g., Chart.js) to display stock price trends.  
**Time**: 6 minutes  
**Skills Tested**: Data visualization, library integration.  
**Prompt**:  
"Create a `StockChart` component that uses Chart.js to display a line chart of stock prices over time."

**Sample Code**:
```jsx
import React, { useEffect, useRef } from 'react';
import Chart from 'chart.js/auto';

interface PricePoint {
  time: string;
  price: number;
}

const StockChart: React.FC<{ data: PricePoint[] }> = ({ data }) => {
  const canvasRef = useRef<HTMLCanvasElement>(null);
  const chartRef = useRef<Chart | null>(null);

  useEffect(() => {
    if (canvasRef.current) {
      chartRef.current = new Chart(canvasRef.current, {
        type: 'line',
        data: {
          labels: data.map((point) => point.time),
          datasets: [
            {
              label: 'Stock Price',
              data: data.map((point) => point.price),
              borderColor: 'blue',
              fill: false,
            },
          ],
        },
      });
    }

    return () => {
      chartRef.current?.destroy();
    };
  }, [data]);

  return <canvas ref={canvasRef} />;
};

export default StockChart;
```

**Evaluation**:  
- **Key Points**: Initialize chart, update with new data, clean up on unmount. Bonus for TypeScript or dynamic updates.  
- **Bloomberg Relevance**: Charts are central to data visualization.  
**Evaluation Criteria**:  
- Chart initialization and rendering (2 points).  
- Data binding (2 points).  
- Cleanup logic (1 point).

#### 9. Handle Concurrent API Requests
**Task**: Write a function to fetch data from multiple APIs concurrently and merge the results.  
**Time**: 6 minutes  
**Skills Tested**: API integration, asynchronous programming.  
**Prompt**:  
"Write a function that fetches stock data and news from two APIs concurrently and combines the results into a single state update."

**Sample Code**:
```jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

interface StockData {
  symbol: string;
  price: number;
}

interface NewsItem {
  title: string;
}

const DataFetcher: React.FC<{ symbol: string }> = ({ symbol }) => {
  const [data, setData] = useState<{ stock: StockData | null; news: NewsItem[] }>({
    stock: null,
    news: [],
  });

  useEffect(() => {
    const fetchData = async () => {
      try {
        const [stockResponse, newsResponse] = await Promise.all([
          axios.get(`https://api.example.com/stocks/${symbol}`),
          axios.get(`https://api.example.com/news/${symbol}`),
        ]);
        setData({
          stock: stockResponse.data,
          news: newsResponse.data,
        });
      } catch (error) {
        console.error('Error fetching data:', error);
      }
    };

    fetchData();
  }, [symbol]);

  return (
    <div>
      <h3>{data.stock?.symbol}</h3>
      <p>Price: ${data.stock?.price.toFixed(2)}</p>
      <ul>
        {data.news.map((item, index) => (
          <li key={index}>{item.title}</li>
        ))}
      </ul>
    </div>
  );
};

export default DataFetcher;
```

**Evaluation**:  
- **Key Points**: Use `Promise.all` for concurrent requests, handle errors. Bonus for TypeScript or caching.  
- **Bloomberg Relevance**: Fetches data from multiple sources simultaneously.  
**Evaluation Criteria**:  
- Concurrent API calls (2 points).  
- Error handling (2 points).  
- State integration (1 point).

#### 10. Discuss Testing Strategy for a Complex UI
**Task**: Explain a testing strategy for a Bloomberg Terminal-like React application.  
**Time**: 5 minutes  
**Skills Tested**: Testing, system design.  
**Prompt**:  
"Describe how you would test a React application with multiple panels, real-time data, and complex interactions. Include tools and types of tests."

**Sample Answer**:  
- **Unit Testing**: Use Jest and React Testing Library to test components (e.g., `StockTicker`, `DataTable`). Test props, state changes, event handlers.  
  - Example: `it('renders stock price correctly', () => { render(<StockTicker symbol="AAPL" />); expect(screen.getByText(/AAPL/)).toBeInTheDocument(); });`  
- **Integration Testing**: Test interactions between components (e.g., panel resizing affecting chart updates).  
- **End-to-End Testing**: Use Cypress to simulate user workflows (e.g., navigating panels, fetching data).  
- **Snapshot Testing**: Capture component outputs to detect unintended changes.  
- **Performance Testing**: Use React Profiler to identify slow renders.  
- **Tools**: Jest, React Testing Library, Cypress, React Profiler.  
- **Strategy**: Prioritize unit tests for components, integration tests for interactions, E2E tests for user flows. Mock APIs/WebSockets for consistency.  

**Evaluation**:  
- **Key Points**: Cover unit, integration, E2E testing, mention tools and scenarios. Bonus for performance or accessibility testing.  
- **Bloomberg Relevance**: Robust testing ensures reliability.  
**Evaluation Criteria**:  
- Comprehensive testing strategy (2 points).  
- Specific tools and examples (2 points).  
- Real-time data or performance considerations (1 point).

---

### Set 2: Security, Debugging, and Production-Grade Practices (30 Minutes)

#### 11. Implement Secure API Authentication
**Task**: Write a React component that securely fetches data from an API using OAuth 2.0 token-based authentication.  
**Time**: 6 minutes  
**Skills Tested**: Security, API integration, error handling.  
**Prompt**:  
"Create a `SecureDataFetcher` component that fetches stock data from an API using OAuth 2.0 token authentication. Handle token expiration and refresh securely."

**Sample Code**:
```jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

interface StockData {
  symbol: string;
  price: number;
}

const SecureDataFetcher: React.FC<{ symbol: string }> = ({ symbol }) => {
  const [data, setData] = useState<StockData | null>(null);
  const [error, setError] = useState<string | null>(null);

  const getAccessToken = async () => {
    try {
      // Mock token fetch
      const response = await axios.post('https://api.example.com/oauth/token', {
        grant_type: 'client_credentials',
        client_id: process.env.REACT_APP_CLIENT_ID,
        client_secret: process.env.REACT_APP_CLIENT_SECRET,
      });
      return response.data.access_token;
    } catch (err) {
      throw new Error('Failed to refresh token');
    }
  };

  useEffect(() => {
    const fetchData = async () => {
      try {
        const token = await getAccessToken();
        const response = await axios.get(`https://api.example.com/stocks/${symbol}`, {
          headers: { Authorization: `Bearer ${token}` },
        });
        setData(response.data);
        setError(null);
      } catch (err) {
        setError('Failed to fetch data');
      }
    };

    fetchData();
  }, [symbol]);

  if (error) return <div>Error: {error}</div>;
  if (!data) return <div>Loading...</div>;

  return (
    <div>
      <h3>{data.symbol}</h3>
      <p>Price: ${data.price.toFixed(2)}</p>
    </div>
  );
};

export default SecureDataFetcher;
```

**Evaluation**:  
- **Key Points**: Securely handle tokens (e.g., avoid state storage, use environment variables), implement error handling, authenticated API calls. Bonus for token storage or refresh logic.  
- **Bloomberg Relevance**: Secure API access protects sensitive financial data.  
**Evaluation Criteria**:  
- Secure token handling (2 points).  
- API integration (2 points).  
- Error handling (1 point).

#### 12. Debug a Performance Issue in a React Component
**Task**: Identify and fix a performance issue in a provided React component that re-renders excessively.  
**Time**: 6 minutes  
**Skills Tested**: Debugging, performance optimization.  
**Prompt**:  
"Given a `StockList` component that re-renders unnecessarily, identify the issue and propose a fix. The component displays a list of stocks and updates when new data arrives."

**Sample Problematic Code**:
```jsx
const StockList = ({ stocks }) => {
  console.log('Rendering StockList');
  return (
    <div>
      {stocks.map((stock) => (
        <div key={stock.id}>
          {stock.symbol}: ${stock.price}
        </div>
      ))}
    </div>
  );
};
```

**Sample Answer**:  
- **Issue**: Component re-renders on every parent render, even if `stocks` hasn’t changed, due to missing `React.memo`. New array references trigger re-renders.  
- **Fix**:
```jsx
import React, { memo } from 'react';

const StockList = memo(({ stocks }) => {
  console.log('Rendering StockList');
  return (
    <div>
      {stocks.map((stock) => (
        <div key={stock.id}>
          {stock.symbol}: ${stock.price}
        </div>
      ))}
    </div>
  );
});
```
- **Explanation**: Use `React.memo` to memoize, ensure parent passes stable `stocks` reference (e.g., via `useMemo`). Check console logs to confirm reduced renders.

**Evaluation**:  
- **Key Points**: Identify re-render issue, propose `React.memo`, suggest stabilizing props. Bonus for mentioning React DevTools Profiler.  
- **Bloomberg Relevance**: Performance is critical for data-heavy UIs.  
**Evaluation Criteria**:  
- Issue identification (2 points).  
- Fix implementation (2 points).  
- Debugging approach (1 point).

#### 13. Ensure Accessibility in a Form Component
**Task**: Enhance a React form component to meet WCAG 2.1 accessibility standards.  
**Time**: 6 minutes  
**Skills Tested**: Accessibility, UI design.  
**Prompt**:  
"Modify a `SearchForm` component to be accessible, ensuring it meets WCAG 2.1 standards (e.g., keyboard navigation, screen reader support)."

**Sample Original Code**:
```jsx
const SearchForm = ({ onSearch }) => {
  return (
    <div>
      <input type="text" placeholder="Search stocks" />
      <button onClick={() => onSearch()}>Search</button>
    </div>
  );
};
```

**Sample Enhanced Code**:
```jsx
import React, { useRef } from 'react';

const SearchForm: React.FC<{ onSearch: (value: string) => void }> = ({ onSearch }) => {
  const inputRef = useRef<HTMLInputElement>(null);

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();
    if (inputRef.current) {
      onSearch(inputRef.current.value);
    }
  };

  return (
    <form onSubmit={handleSubmit} aria-label="Stock search form">
      <label htmlFor="search-input" className="sr-only">
        Search stocks
      </label>
      <input
        id="search-input"
        type="text"
        ref={inputRef}
        placeholder="Search stocks"
        aria-required="true"
        onKeyDown={(e) => e.key === 'Enter' && handleSubmit(e)}
      />
      <button type="submit" aria-label="Submit search">
        Search
      </button>
    </form>
  );
};

export default SearchForm;
```

**Evaluation**:  
- **Key Points**: Add ARIA attributes, ensure keyboard navigation, use semantic HTML. Bonus for screen reader testing or focus management.  
- **Bloomberg Relevance**: Accessibility ensures usability for all users.  
**Evaluation Criteria**:  
- ARIA and semantic HTML (2 points).  
- Keyboard support (2 points).  
- Accessibility explanation (1 point).

#### 14. Implement Error Boundary for Component Resilience
**Task**: Create an error boundary component to handle runtime errors in a chart component.  
**Time**: 6 minutes  
**Skills Tested**: Error handling, production readiness.  
**Prompt**:  
"Write an `ErrorBoundary` component to catch errors in a `StockChart` component and display a fallback UI."

**Sample Code**:
```jsx
import React, { Component, ReactNode } from 'react';

interface ErrorBoundaryState {
  hasError: boolean;
}

class ErrorBoundary extends Component<{ children: ReactNode }, ErrorBoundaryState> {
  state: ErrorBoundaryState = { hasError: false };

  static getDerivedStateFromError() {
    return { hasError: true };
  }

  componentDidCatch(error: Error, info: React.ErrorInfo) {
    console.error('Error caught:', error, info);
    // Log to monitoring service (e.g., Sentry)
  }

  render() {
    if (this.state.hasError) {
      return (
        <div role="alert">
          <h3>Something went wrong</h3>
          <button onClick={() => this.setState({ hasError: false })}>Retry</button>
        </div>
      );
    }
    return this.props.children;
  }
}

// Usage
const App = () => (
  <ErrorBoundary>
    <StockChart />
  </ErrorBoundary>
);

export default ErrorBoundary;
```

**Evaluation**:  
- **Key Points**: Implement `getDerivedStateFromError` and `componentDidCatch`, provide fallback UI, suggest logging. Bonus for retry logic or monitoring integration.  
- **Bloomberg Relevance**: Error boundaries ensure resilience during component failures.  
**Evaluation Criteria**:  
- Error boundary implementation (2 points).  
- Fallback UI (2 points).  
- Logging/retry logic (1 point).

#### 15. Secure WebSocket Communication
**Task**: Explain and code a secure WebSocket connection for real-time stock updates.  
**Time**: 6 minutes  
**Skills Tested**: Security, WebSocket integration.  
**Prompt**:  
"Describe how to secure a WebSocket connection for real-time stock data and provide a code snippet to initialize it securely in a React component."

**Sample Answer/Code**:
```jsx
import React, { useState, useEffect } from 'react';

interface StockData {
  symbol: string;
  price: number;
}

const SecureStockTicker: React.FC<{ symbol: string }> = ({ symbol }) => {
  const [data, setData] = useState<StockData | null>(null);

  useEffect(() => {
    // Secure WebSocket with wss:// and token
    const ws = new WebSocket('wss://secure-stock-api.com?token=' + encodeURIComponent(process.env.REACT_APP_WS_TOKEN));

    ws.onopen = () => {
      console.log('WebSocket connected securely');
      ws.send(JSON.stringify({ symbol }));
    };

    ws.onmessage = (event) => {
      const stockData: StockData = JSON.parse(event.data);
      setData(stockData);
    };

    ws.onerror = (error) => {
      console.error('WebSocket error:', error);
    };

    return () => {
      ws.close();
    };
  }, [symbol]);

  return data ? (
    <div>
      <h3>{data.symbol}</h3>
      <p>Price: ${data.price.toFixed(2)}</p>
    </div>
  ) : (
    <div>Loading...</div>
  );
};

export default SecureStockTicker;
```

**Evaluation**:  
- **Key Points**: Use `wss://` for encryption, include token-based authentication, handle errors. Explain security measures (e.g., TLS, origin validation). Bonus for rate limiting or message validation.  
- **Bloomberg Relevance**: Secure real-time data is critical for financial apps.  
**Evaluation Criteria**:  
- Secure WebSocket setup (2 points).  
- Authentication (2 points).  
- Error handling (1 point).

---

### Set 3: Complementary Tech Stacks for Modern React Applications (30 Minutes)

#### 16. TypeScript Integration for a Typed Component
**Task**: Convert a React component to TypeScript, ensuring type safety for props and state.  
**Time**: 6 minutes  
**Skills Tested**: TypeScript, type safety.  
**Prompt**:  
"Convert a `StockCard` component to TypeScript, defining types for props and state, and handle a nullable data prop."

**Sample Code**:
```tsx
import React, { useState } from 'react';

interface Stock {
  symbol: string;
  price: number;
}

interface StockCardProps {
  stock: Stock | null;
  onSelect: (symbol: string) => void;
}

const StockCard: React.FC<StockCardProps> = ({ stock, onSelect }) => {
  const [isHovered, setIsHovered] = useState<boolean>(false);

  if (!stock) return <div>No data available</div>;

  return (
    <div
      onMouseEnter={() => setIsHovered(true)}
      onMouseLeave={() => setIsHovered(false)}
      onClick={() => onSelect(stock.symbol)}
      style={{ border: isHovered ? '2px solid blue' : '1px solid gray', padding: '10px' }}
    >
      <h3>{stock.symbol}</h3>
      <p>Price: ${stock.price.toFixed(2)}</p>
    </div>
  );
};

export default StockCard;
```

**Evaluation**:  
- **Key Points**: Define interfaces for props/state, handle nullable props, ensure type-safe handlers. Bonus for discriminated unions or generics.  
- **Bloomberg Relevance**: TypeScript ensures maintainability in large-scale apps.  
**Evaluation Criteria**:  
- Type definitions (2 points).  
- Nullable handling (2 points).  
- Event typing (1 point).

#### 17. Fetch Data with GraphQL
**Task**: Write a React component that fetches stock data using GraphQL with Apollo Client.  
**Time**: 6 minutes  
**Skills Tested**: GraphQL, Apollo Client, API integration.  
**Prompt**:  
"Create a `StockDetails` component that uses Apollo Client to fetch stock data via a GraphQL query."

**Sample Code**:
```jsx
import React from 'react';
import { useQuery, gql } from '@apollo/client';

const STOCK_QUERY = gql`
  query GetStock($symbol: String!) {
    stock(symbol: $symbol) {
      symbol
      price
    }
  }
`;

interface Stock {
  symbol: string;
  price: number;
}

const StockDetails: React.FC<{ symbol: string }> = ({ symbol }) => {
  const { data, loading, error } = useQuery<{ stock: Stock }>(STOCK_QUERY, {
    variables: { symbol },
  });

  if (loading) return <div>Loading...</div>;
  if (error) return <div>Error: {error.message}</div>;

  return (
    <div>
      <h3>{data?.stock.symbol}</h3>
      <p>Price: ${data?.stock.price.toFixed(2)}</p>
    </div>
  );
};

export default StockDetails;
```

**Evaluation**:  
- **Key Points**: Write valid GraphQL query, use `useQuery`, handle loading/error states. Bonus for TypeScript or caching.  
- **Bloomberg Relevance**: GraphQL is efficient for complex financial data.  
**Evaluation Criteria**:  
- GraphQL query (2 points).  
- Apollo integration (2 points).  
- State handling (1 point).

#### 18. Server-Side Rendering with Next.js
**Task**: Create a Next.js page that server-side renders a stock dashboard.  
**Time**: 6 minutes  
**Skills Tested**: Next.js, server-side rendering.  
**Prompt**:  
"Write a Next.js page that fetches stock data server-side and renders a dashboard with a `StockCard` component."

**Sample Code**:
```jsx
import { GetServerSideProps } from 'next';
import StockCard from '../components/StockCard';

interface Stock {
  symbol: string;
  price: number;
}

interface StockDashboardProps {
  stocks: Stock[];
}

const StockDashboard: React.FC<StockDashboardProps> = ({ stocks }) => {
  return (
    <div>
      <h1>Stock Dashboard</h1>
      {stocks.map((stock) => (
        <StockCard key={stock.id} stock={stock} onSelect={(symbol) => console.log(symbol)} />
      ))}
    </div>
  );
};

export const getServerSideProps: GetServerSideProps = async () => {
  // Mock API call
  const stocks = [
    { symbol: 'AAPL', price: 150.25 },
    { symbol: 'GOOGL', price: 2800.75 },
  ];
  return {
    props: { stocks },
  };
};

export default StockDashboard;
```

**Evaluation**:  
- **Key Points**: Use `getServerSideProps` to fetch data, pass to component. Bonus for SEO or hydration discussion.  
- **Bloomberg Relevance**: SSR improves initial load time for dashboards.  
**Evaluation Criteria**:  
- SSR implementation (2 points).  
- Data fetching (2 points).  
- Component integration (1 point).

#### 19. Offload Heavy Computations to Web Workers
**Task**: Use a Web Worker to perform a heavy computation (e.g., calculating moving averages) in a React app.  
**Time**: 6 minutes  
**Skills Tested**: Web Workers, performance optimization.  
**Prompt**:  
"Write a React component that offloads the calculation of a 50-day moving average for stock prices to a Web Worker."

**Sample Code**:
```jsx
// worker.js
self.onmessage = (e) => {
  const prices = e.data;
  const movingAverage = prices
    .slice(-50)
    .reduce((sum, price, i, arr) => sum + price / arr.length, 0);
  self.postMessage(movingAverage);
};

// StockMovingAverage.tsx
import React, { useState, useEffect } from 'react';

const StockMovingAverage: React.FC<{ prices: number[] }> = ({ prices }) => {
  const [average, setAverage] = useState<number | null>(null);

  useEffect(() => {
    const worker = new Worker(new URL('./worker.js', import.meta.url));
    worker.postMessage(prices);

    worker.onmessage = (e) => {
      setAverage(e.data);
    };

    return () => worker.terminate();
  }, [prices]);

  return average ? (
    <div>50-Day Moving Average: ${average.toFixed(2)}</div>
  ) : (
    <div>Calculating...</div>
  );
};

export default StockMovingAverage;
```

**Evaluation**:  
- **Key Points**: Create Web Worker, offload computation, handle communication. Bonus for error handling or cleanup.  
- **Bloomberg Relevance**: Web Workers prevent UI blocking in data-intensive apps.  
**Evaluation Criteria**:  
- Worker setup (2 points).  
- Computation offloading (2 points).  
- Cleanup (1 point).

#### 20. Integrate D3.js for Custom Data Visualization
**Task**: Create a React component that uses D3.js to render a custom stock price sparkline.  
**Time**: 6 minutes  
**Skills Tested**: D3.js, data visualization.  
**Prompt**:  
"Write a `StockSparkline` component that uses D3.js to render a simple sparkline of stock prices."

**Sample Code**:
```jsx
import React, { useEffect, useRef } from 'react';
import * as d3 from 'd3';

interface PricePoint {
  time: string;
  price: number;
}

const StockSparkline: React.FC<{ data: PricePoint[] }> = ({ data }) => {
  const svgRef = useRef<SVGSVGElement>(null);

  useEffect(() => {
    if (svgRef.current) {
      const svg = d3.select(svgRef.current);
      const width = 200;
      const height = 50;

      const x = d3
        .scaleTime()
        .domain(d3.extent(data, (d) => new Date(d.time)) as [Date, Date])
        .range([0, width]);

      const y = d3
        .scaleLinear()
        .domain(d3.extent(data, (d) => d.price) as [number, number])
        .range([height, 0]);

      const line = d3
        .line<PricePoint>()
        .x((d) => x(new Date(d.time)))
        .y((d) => y(d.price));

      svg.selectAll('*').remove();
      svg
        .append('path')
        .datum(data)
        .attr('fill', 'none')
        .attr('stroke', 'blue')
        .attr('d', line);
    }
  }, [data]);

  return <svg ref={svgRef} width={200} height={50} />;
};

export default StockSparkline;
```

**Evaluation**:  
- **Key Points**: Use D3.js for sparkline, manage SVG rendering, update with new data. Bonus for TypeScript or optimizations.  
- **Bloomberg Relevance**: Custom visualizations are common in financial dashboards.  
**Evaluation Criteria**:  
- D3.js integration (2 points).  
- Sparkline rendering (2 points).  
- Data updates (1 point).

---

### Evaluation Scoring
Each task is scored out of 5 points, totaling 100 points across 20 tasks. A score of 80+ indicates a strong candidate capable of building a production-grade, Bloomberg Terminal-like application.  
- **Set 1 (Core React)**: Focus on React proficiency, real-time data, and UI design.  
- **Set 2 (Security/Debugging)**: Assess secure data handling, error management, accessibility.  
- **Set 3 (Complementary Tech)**: Evaluate TypeScript, GraphQL, Next.js, Web Workers, D3.js.

**Key Evaluation Focus**:  
- **Code Quality**: Clean, modular, type-safe code.  
- **Performance**: Optimization techniques (e.g., memoization, virtualization).  
- **Real-Time Handling**: Proper WebSocket/API management.  
- **UX Design**: Modular, accessible layouts.  
- **Security**: Secure communication and authentication.  
- **Scalability**: Solutions for large-scale applications.

**Additional Notes**:  
- **Time Management**: Allocate ~5-6 minutes per task. Candidates can write partial code, sketch designs, or explain approaches for discussion tasks.  
- **Bloomberg Terminal Context**: Emphasizes real-time data, modular layouts, performance, and security.  
- **Execution**: Provide a coding setup (e.g., CodeSandbox) for coding tasks. Encourage whiteboarding or verbal explanations for discussions.  
- **Customization**: If specific requirements (e.g., libraries, backend integrations) are needed, the tasks can be adjusted.

---

This completes the full 20 assessments with all details preserved. You can copy this text into a document editor (e.g., Microsoft Word, Google Docs), format as needed (e.g., add headings, adjust styling), and export to PDF for interviewer reference. If you need specific formatting suggestions or further refinements, please let me know!