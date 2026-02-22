# Behavioral Questions & DSA Patterns - Interview Q&A

> STAR method stories from your experience + coding pattern cheat sheets

---

## Table of Contents

- [Your Pitch & Stories (STAR Method)](#your-pitch--stories-star-method)
- [Common Behavioral Questions](#common-behavioral-questions)
- [DSA Patterns Cheat Sheet](#dsa-patterns-cheat-sheet)
- [Quick Reference Card](#quick-reference-card)

---

## Your Pitch & Stories (STAR Method)

### Q1: Tell me about yourself (2-minute pitch).

**Answer:**
> "I'm Awais, a Senior Software Engineer with over 5 years of experience building scalable full-stack applications. I specialize in the MERN stack, AWS cloud architecture, and Web3 development on Solana.
>
> Currently at a stealth startup, I've been building AI-powered trading platforms with auto-strategy execution, deploying CVAT annotation workflows with YOLO integration on AWS, and designing event-driven architectures using SNS+SQS fan-out patterns — which improved our system efficiency by 30%.
>
> Before that, at Codora, I led blockchain integration across both Ethereum and Solana ecosystems, working with everything from SPL tokens and DeFi protocols to building the frontend with Next.js and backend with NestJS microservices.
>
> I graduated from NUST with a BS in Computer Science. What excites me most is the intersection of AI, blockchain, and cloud-native architecture. I'm looking for a senior role where I can leverage this unique combination to build impactful products at scale, while mentoring the team and driving technical decisions."

---

### Q2: Tell me about the most challenging technical problem you solved.

**Answer (STAR):**
> **Situation:** At my current startup, our trading platform was processing trade events sequentially through a single pipeline. During peak trading hours, notifications, analytics, and audit logs were all competing for processing time, causing delays of 5-10 seconds.
>
> **Task:** I needed to redesign the event processing architecture to handle 10x throughput without losing messages or adding significant latency, while maintaining reliability.
>
> **Action:** I designed and implemented an SNS + SQS fan-out pattern. I published trade events to an SNS topic that fanned out to dedicated SQS queues — one each for analytics, notifications, risk checks, and audit logging. Each consumer could scale independently via ECS auto-scaling. I added DLQs for fault tolerance and CloudWatch alarms for monitoring. I also implemented SNS message filtering so queues only received relevant events.
>
> **Result:** System efficiency improved by 30%. Message processing became parallel instead of sequential. Each service now scales independently based on its specific load. The system handles 10x the original volume with sub-second latency, and the DLQ monitoring has caught and allowed us to fix issues before they impacted users.

---

### Q3: Tell me about a time you led a team or project.

**Answer (STAR):**
> **Situation:** At Codora, we needed to integrate blockchain functionality (Ethereum + Solana) into our existing web platform. The team had no blockchain experience.
>
> **Task:** I was asked to lead the blockchain integration effort, including architecture decisions, implementation, and upskilling the team.
>
> **Action:** I started by researching and creating a technical spec comparing implementation approaches. I chose Solana for our primary chain due to speed and cost. I set up the Anchor framework, created reusable wallet integration components with Privy, and established indexing using Helius webhooks and Geyser gRPC. I paired with team members on their first blockchain features and created internal documentation.
>
> **Result:** We delivered 5 blockchain projects, reducing development time by 15% through the reusable components I built. Two junior developers went from zero blockchain knowledge to independently building Solana integrations within 3 months.

---

### Q4: Tell me about a time you disagreed with a teammate.

**Answer (STAR):**
> **Situation:** At Codora, a senior colleague wanted to build a custom WebSocket server for real-time features. I believed we should use Pusher (managed service).
>
> **Task:** We needed to reach a decision quickly as the sprint was starting.
>
> **Action:** Instead of arguing opinions, I suggested we spend 2 hours each creating a comparison document. I listed: development time, maintenance cost, scaling complexity, team expertise, and risk. We found that custom WebSocket would take 3 weeks to build properly (handling reconnection, load balancing, monitoring), while Pusher could be integrated in 2 days with built-in features.
>
> **Result:** We went with Pusher. It was integrated in 2 days, saved approximately 2.5 weeks of development, and the team could focus on business logic instead of infrastructure. My colleague later agreed it was the right call given our timeline and team size. The key lesson was using data over opinions to make technical decisions.

---

### Q5: Tell me about a time you dealt with a production incident.

**Answer (STAR):**
> **Situation:** At my current startup, our trading platform experienced a sudden spike in 500 errors on a Friday evening. Users couldn't execute trades.
>
> **Task:** I needed to diagnose and fix the issue quickly to minimize downtime and financial impact.
>
> **Action:** I checked our monitoring (New Relic + CloudWatch). The error logs showed PostgreSQL connection pool exhaustion. I traced it to a recent deployment that introduced a query without proper connection release in an edge case. I immediately: (1) rolled back the deployment via ECS, (2) identified the exact code issue — a missing `finally` block in a transaction, (3) fixed the code, added a connection pool health check, and deployed the fix.
>
> **Result:** Total downtime was 12 minutes. I then added connection pool monitoring to our CloudWatch dashboards, set up an alarm for pool exhaustion, and implemented a code review checklist item for database connection handling. The platform maintained 99.9% uptime for the quarter.

---

### Q6: How do you handle working with tight deadlines?

**Answer:**
> "I focus on three things:
>
> 1. **Prioritize ruthlessly** — I identify the MVP (minimum viable product) and cut anything that's nice-to-have. I communicate clearly with stakeholders about what will and won't make the deadline.
>
> 2. **Reduce risk early** — I tackle the hardest/riskiest parts first. If something is going to block us, I want to know on day 1, not day 10.
>
> 3. **Communicate proactively** — If I see we're going to miss the deadline, I raise it immediately with options: 'We can ship feature A and B by Friday, or all three by next Wednesday. Which do you prefer?'
>
> For example, at Merik Solutions, we had a client demo in 2 weeks for a feature originally scoped for 4 weeks. I broke it into must-have and nice-to-have, parallelized work across the team, and we delivered the core functionality on time. The nice-to-haves shipped the following week."

---

### Q7: How do you approach code reviews?

**Answer:**
> "I see code reviews as a teaching and quality tool, not a gatekeeping exercise.
>
> **What I look for:**
> 1. **Correctness** — Does it actually solve the problem? Edge cases?
> 2. **Security** — SQL injection, XSS, secrets in code
> 3. **Performance** — N+1 queries, unnecessary re-renders, missing indexes
> 4. **Maintainability** — Is it readable? Would a new team member understand it?
> 5. **Tests** — Are critical paths tested?
>
> **How I give feedback:**
> - I always explain *why* something should change, not just *what*
> - I differentiate between blocking issues (security, bugs) and suggestions (style, naming)
> - I highlight things done well, not just problems
> - For complex changes, I suggest pair-programming instead of back-and-forth comments"

---

### Q8: Why are you looking for a new role?

**Answer:**
> "I'm looking for a role where I can have more impact at a company that's scaling. My experience across MERN, AWS, Web3, and AI gives me a unique perspective, and I want to apply that at a company where these technologies intersect with real business problems. I'm particularly excited about [company-specific reason] and the opportunity to [mentor/lead/build something specific]."

---

## Common Behavioral Questions

**Prepare STAR stories for these:**

| Question | Story to Use |
|---|---|
| Tell me about a failure | Production incident / Early architecture decision that needed refactoring |
| How do you mentor? | Training juniors on blockchain at Codora |
| Biggest achievement? | SNS+SQS redesign (30% improvement) |
| Working with ambiguity? | Stealth startup — defining architecture from scratch |
| Cross-team collaboration? | Integrating with data team for CVAT/YOLO pipeline |

---

## DSA Patterns Cheat Sheet

### Pattern 1: Two Pointers

```javascript
// Remove duplicates from sorted array
function removeDuplicates(nums) {
  let slow = 0;
  for (let fast = 1; fast < nums.length; fast++) {
    if (nums[fast] !== nums[slow]) {
      slow++;
      nums[slow] = nums[fast];
    }
  }
  return slow + 1;
}

// Container with most water
function maxArea(height) {
  let left = 0, right = height.length - 1, max = 0;
  while (left < right) {
    const area = Math.min(height[left], height[right]) * (right - left);
    max = Math.max(max, area);
    if (height[left] < height[right]) left++;
    else right--;
  }
  return max;
}

// 3Sum
function threeSum(nums) {
  nums.sort((a, b) => a - b);
  const result = [];
  for (let i = 0; i < nums.length - 2; i++) {
    if (i > 0 && nums[i] === nums[i - 1]) continue; // skip duplicates
    let left = i + 1, right = nums.length - 1;
    while (left < right) {
      const sum = nums[i] + nums[left] + nums[right];
      if (sum === 0) {
        result.push([nums[i], nums[left], nums[right]]);
        while (left < right && nums[left] === nums[left + 1]) left++;
        while (left < right && nums[right] === nums[right - 1]) right--;
        left++; right--;
      } else if (sum < 0) left++;
      else right--;
    }
  }
  return result;
}
```

---

### Pattern 2: Sliding Window

```javascript
// Maximum sum subarray of size k
function maxSubarraySum(arr, k) {
  let windowSum = arr.slice(0, k).reduce((a, b) => a + b);
  let maxSum = windowSum;
  for (let i = k; i < arr.length; i++) {
    windowSum += arr[i] - arr[i - k];
    maxSum = Math.max(maxSum, windowSum);
  }
  return maxSum;
}

// Longest substring without repeating characters
function lengthOfLongestSubstring(s) {
  const seen = new Map();
  let start = 0, maxLen = 0;
  for (let end = 0; end < s.length; end++) {
    if (seen.has(s[end]) && seen.get(s[end]) >= start) {
      start = seen.get(s[end]) + 1;
    }
    seen.set(s[end], end);
    maxLen = Math.max(maxLen, end - start + 1);
  }
  return maxLen;
}

// Minimum window substring
function minWindow(s, t) {
  const need = new Map();
  for (const c of t) need.set(c, (need.get(c) || 0) + 1);

  let have = 0, required = need.size;
  let left = 0, minLen = Infinity, result = '';

  for (let right = 0; right < s.length; right++) {
    const c = s[right];
    need.set(c, (need.get(c) || 0) - 1);
    if (need.get(c) === 0) have++;

    while (have === required) {
      if (right - left + 1 < minLen) {
        minLen = right - left + 1;
        result = s.slice(left, right + 1);
      }
      need.set(s[left], need.get(s[left]) + 1);
      if (need.get(s[left]) > 0) have--;
      left++;
    }
  }
  return result;
}
```

---

### Pattern 3: Binary Search

```javascript
// Standard binary search
function binarySearch(nums, target) {
  let left = 0, right = nums.length - 1;
  while (left <= right) {
    const mid = Math.floor((left + right) / 2);
    if (nums[mid] === target) return mid;
    if (nums[mid] < target) left = mid + 1;
    else right = mid - 1;
  }
  return -1;
}

// Search in rotated sorted array
function search(nums, target) {
  let left = 0, right = nums.length - 1;
  while (left <= right) {
    const mid = Math.floor((left + right) / 2);
    if (nums[mid] === target) return mid;

    // Left half is sorted
    if (nums[left] <= nums[mid]) {
      if (target >= nums[left] && target < nums[mid]) right = mid - 1;
      else left = mid + 1;
    }
    // Right half is sorted
    else {
      if (target > nums[mid] && target <= nums[right]) left = mid + 1;
      else right = mid - 1;
    }
  }
  return -1;
}
```

---

### Pattern 4: BFS & DFS

```javascript
// BFS - Level order traversal
function levelOrder(root) {
  if (!root) return [];
  const result = [], queue = [root];
  while (queue.length) {
    const level = [];
    const size = queue.length;
    for (let i = 0; i < size; i++) {
      const node = queue.shift();
      level.push(node.val);
      if (node.left) queue.push(node.left);
      if (node.right) queue.push(node.right);
    }
    result.push(level);
  }
  return result;
}

// DFS - Max depth of binary tree
function maxDepth(root) {
  if (!root) return 0;
  return 1 + Math.max(maxDepth(root.left), maxDepth(root.right));
}

// Validate BST
function isValidBST(root, min = -Infinity, max = Infinity) {
  if (!root) return true;
  if (root.val <= min || root.val >= max) return false;
  return isValidBST(root.left, min, root.val) && isValidBST(root.right, root.val, max);
}

// Number of islands (DFS on grid)
function numIslands(grid) {
  let count = 0;
  for (let i = 0; i < grid.length; i++) {
    for (let j = 0; j < grid[0].length; j++) {
      if (grid[i][j] === '1') {
        count++;
        dfs(grid, i, j);
      }
    }
  }
  return count;
}

function dfs(grid, i, j) {
  if (i < 0 || j < 0 || i >= grid.length || j >= grid[0].length || grid[i][j] === '0') return;
  grid[i][j] = '0'; // mark visited
  dfs(grid, i + 1, j);
  dfs(grid, i - 1, j);
  dfs(grid, i, j + 1);
  dfs(grid, i, j - 1);
}
```

---

### Pattern 5: Dynamic Programming

```javascript
// Climbing stairs (Fibonacci variant)
function climbStairs(n) {
  if (n <= 2) return n;
  let prev2 = 1, prev1 = 2;
  for (let i = 3; i <= n; i++) {
    [prev2, prev1] = [prev1, prev2 + prev1];
  }
  return prev1;
}

// Coin change (minimum coins)
function coinChange(coins, amount) {
  const dp = new Array(amount + 1).fill(Infinity);
  dp[0] = 0;
  for (let i = 1; i <= amount; i++) {
    for (const coin of coins) {
      if (coin <= i) {
        dp[i] = Math.min(dp[i], dp[i - coin] + 1);
      }
    }
  }
  return dp[amount] === Infinity ? -1 : dp[amount];
}

// Longest Common Subsequence
function longestCommonSubsequence(text1, text2) {
  const dp = Array(text1.length + 1).fill(null).map(() =>
    Array(text2.length + 1).fill(0)
  );
  for (let i = 1; i <= text1.length; i++) {
    for (let j = 1; j <= text2.length; j++) {
      if (text1[i - 1] === text2[j - 1]) {
        dp[i][j] = dp[i - 1][j - 1] + 1;
      } else {
        dp[i][j] = Math.max(dp[i - 1][j], dp[i][j - 1]);
      }
    }
  }
  return dp[text1.length][text2.length];
}

// 0/1 Knapsack
function knapsack(weights, values, capacity) {
  const n = weights.length;
  const dp = Array(n + 1).fill(null).map(() => Array(capacity + 1).fill(0));

  for (let i = 1; i <= n; i++) {
    for (let w = 1; w <= capacity; w++) {
      if (weights[i - 1] <= w) {
        dp[i][w] = Math.max(
          dp[i - 1][w],
          dp[i - 1][w - weights[i - 1]] + values[i - 1]
        );
      } else {
        dp[i][w] = dp[i - 1][w];
      }
    }
  }
  return dp[n][capacity];
}
```

---

### Pattern 6: Graph Algorithms

```javascript
// Topological Sort (Course Schedule)
function canFinish(numCourses, prerequisites) {
  const graph = Array.from({ length: numCourses }, () => []);
  const inDegree = new Array(numCourses).fill(0);

  for (const [course, prereq] of prerequisites) {
    graph[prereq].push(course);
    inDegree[course]++;
  }

  const queue = [];
  for (let i = 0; i < numCourses; i++) {
    if (inDegree[i] === 0) queue.push(i);
  }

  const order = [];
  while (queue.length) {
    const node = queue.shift();
    order.push(node);
    for (const neighbor of graph[node]) {
      inDegree[neighbor]--;
      if (inDegree[neighbor] === 0) queue.push(neighbor);
    }
  }
  return order.length === numCourses;
}

// Dijkstra's Shortest Path
function dijkstra(graph, start) {
  const dist = new Array(graph.length).fill(Infinity);
  dist[start] = 0;
  const pq = new MinPriorityQueue();
  pq.enqueue(start, 0);

  while (!pq.isEmpty()) {
    const { element: u, priority: d } = pq.dequeue();
    if (d > dist[u]) continue; // skip outdated entries

    for (const [v, weight] of graph[u]) {
      const newDist = dist[u] + weight;
      if (newDist < dist[v]) {
        dist[v] = newDist;
        pq.enqueue(v, newDist);
      }
    }
  }
  return dist;
}

// Union-Find (for connected components, cycle detection)
class UnionFind {
  constructor(n) {
    this.parent = Array.from({ length: n }, (_, i) => i);
    this.rank = new Array(n).fill(0);
  }

  find(x) {
    if (this.parent[x] !== x) {
      this.parent[x] = this.find(this.parent[x]); // path compression
    }
    return this.parent[x];
  }

  union(x, y) {
    const px = this.find(x), py = this.find(y);
    if (px === py) return false; // already connected (cycle!)
    if (this.rank[px] < this.rank[py]) this.parent[px] = py;
    else if (this.rank[px] > this.rank[py]) this.parent[py] = px;
    else { this.parent[py] = px; this.rank[px]++; }
    return true;
  }
}
```

---

### Pattern 7: Heap / Priority Queue

```javascript
// Kth Largest Element
// Use a min-heap of size k
function findKthLargest(nums, k) {
  const minHeap = new MinPriorityQueue();
  for (const num of nums) {
    minHeap.enqueue(num);
    if (minHeap.size() > k) minHeap.dequeue();
  }
  return minHeap.front().element;
}

// Merge K Sorted Lists
function mergeKLists(lists) {
  const pq = new MinPriorityQueue({ priority: (node) => node.val });
  for (const head of lists) {
    if (head) pq.enqueue(head);
  }

  const dummy = new ListNode(0);
  let current = dummy;

  while (!pq.isEmpty()) {
    const node = pq.dequeue().element;
    current.next = node;
    current = current.next;
    if (node.next) pq.enqueue(node.next);
  }

  return dummy.next;
}

// Top K Frequent Elements
function topKFrequent(nums, k) {
  const freq = new Map();
  for (const n of nums) freq.set(n, (freq.get(n) || 0) + 1);

  // Bucket sort approach (O(n))
  const buckets = Array.from({ length: nums.length + 1 }, () => []);
  for (const [num, count] of freq) {
    buckets[count].push(num);
  }

  const result = [];
  for (let i = buckets.length - 1; i >= 0 && result.length < k; i--) {
    result.push(...buckets[i]);
  }
  return result.slice(0, k);
}
```

---

## Quick Reference Card

### Your Key Metrics (Memorize)
| Metric | Number |
|---|---|
| System efficiency improvement | **30%** (SNS+SQS fan-out) |
| Transaction speed improvement | **20%** (AI trading platform) |
| Development time reduction | **15%** (blockchain smart contracts) |
| System uptime | **99.9%** (cloud-native on GCP & AWS) |
| Blockchain projects delivered | **5** |
| Years of experience | **5+** |

### Tech Stack Quick Recall
| Layer | Technologies |
|---|---|
| Frontend | React, Next.js, Tailwind, MUI, Zustand |
| Backend | NestJS, Node.js, Express |
| Database | PostgreSQL, MongoDB, DynamoDB, Redis |
| Cloud | AWS (ECS, SQS, SNS, RDS, S3), GCP |
| Web3 | Solana (Anchor), Ethereum, SPL Tokens |
| AI | LangChain, YOLO, CVAT, OpenAI |
| DevOps | Docker, GitHub Actions, Terraform, Prometheus, New Relic |
| Testing | Cypress, Jest, React Testing Library |

### LeetCode Must-Solve (Top 25)
| # | Problem | Pattern |
|---|---|---|
| 1 | Two Sum | Hash Map |
| 3 | Longest Substring Without Repeating | Sliding Window |
| 15 | 3Sum | Two Pointers |
| 20 | Valid Parentheses | Stack |
| 21 | Merge Two Sorted Lists | Linked List |
| 33 | Search in Rotated Sorted Array | Binary Search |
| 49 | Group Anagrams | Hash Map |
| 53 | Maximum Subarray | DP (Kadane's) |
| 56 | Merge Intervals | Sorting |
| 70 | Climbing Stairs | DP |
| 76 | Minimum Window Substring | Sliding Window |
| 98 | Validate BST | DFS |
| 102 | Level Order Traversal | BFS |
| 121 | Best Time to Buy/Sell Stock | DP |
| 128 | Longest Consecutive Sequence | Hash Set |
| 133 | Clone Graph | DFS/BFS |
| 146 | LRU Cache | Hash Map + Linked List |
| 200 | Number of Islands | DFS/BFS |
| 206 | Reverse Linked List | Linked List |
| 207 | Course Schedule | Topological Sort |
| 226 | Invert Binary Tree | DFS |
| 238 | Product of Array Except Self | Array |
| 322 | Coin Change | DP |
| 347 | Top K Frequent Elements | Heap/Bucket Sort |
| 424 | Longest Repeating Char Replacement | Sliding Window |

---

> **Back to main**: [INTERVIEW_ROADMAP.md](../INTERVIEW_ROADMAP.md)
> **Prev**: [Security & Auth](./10-security-auth.md)
