# Q&A: Prefix Sum and Kadane's Algorithm

## Question

**Explain the concepts of Prefix Sum and Kadane's Algorithm. How are they used to solve array problems efficiently? Provide an example problem that demonstrates both techniques.**

---

## Answer

### 1. Prefix Sum

**Definition:**
Prefix Sum is a technique where we precompute cumulative sums of an array to answer range sum queries in constant time O(1).

**How it works:**
- Create a new array `prefix[]` where `prefix[i]` stores the sum of all elements from index `0` to `i`.
- Formula: `prefix[i] = prefix[i-1] + arr[i]` (with `prefix[0] = arr[0]`)
- To find sum of subarray from index `L` to `R`: `sum = prefix[R] - prefix[L-1]` (if L > 0, else `prefix[R]`)

**Time Complexity:**
- Preprocessing: O(n)
- Query: O(1)

**Use Cases:**
- Range sum queries
- Finding subarrays with a given sum
- Balancing array partitions

---

### 2. Kadane's Algorithm

**Definition:**
Kadane's Algorithm is a dynamic programming approach to find the **Maximum Sum Subarray** in a one-dimensional array in O(n) time.

**How it works:**
- Maintain two variables:
  - `current_max`: Maximum sum ending at current position
  - `global_max`: Maximum sum found so far
- At each element, decide whether to:
  - Extend the existing subarray (`current_max + arr[i]`)
  - Start a new subarray from current element (`arr[i]`)
- Formula: `current_max = max(arr[i], current_max + arr[i])`
- Update `global_max = max(global_max, current_max)`

**Time Complexity:** O(n)  
**Space Complexity:** O(1)

**Use Cases:**
- Maximum sum subarray
- Stock buy/sell problems (maximum profit)
- Maximum product subarray (with modifications)

---

### Example Problem: Maximum Sum Subarray

**Problem Statement:**
Given an integer array `nums`, find the contiguous subarray (containing at least one number) which has the largest sum and return its sum.

**Example:**
```
Input: nums = [-2, 1, -3, 4, -1, 2, 1, -5, 4]
Output: 6
Explanation: The subarray [4, -1, 2, 1] has the largest sum = 6.
```

**Solution using Kadane's Algorithm (JavaScript):**

```javascript
function maxSubArray(nums) {
    let currentMax = nums[0];
    let globalMax = nums[0];
    
    for (let i = 1; i < nums.length; i++) {
        // Either extend existing subarray or start new one
        currentMax = Math.max(nums[i], currentMax + nums[i]);
        // Update global maximum
        globalMax = Math.max(globalMax, currentMax);
    }
    
    return globalMax;
}

// Test
console.log(maxSubArray([-2, 1, -3, 4, -1, 2, 1, -5, 4])); // Output: 6
```

**Solution using Prefix Sum (Alternative Approach):**

```javascript
function maxSubArrayPrefixSum(nums) {
    const n = nums.length;
    const prefix = new Array(n);
    prefix[0] = nums[0];
    
    // Build prefix sum array
    for (let i = 1; i < n; i++) {
        prefix[i] = prefix[i - 1] + nums[i];
    }
    
    let maxSum = nums[0];
    let minPrefix = 0;
    
    for (let i = 0; i < n; i++) {
        // Max subarray ending at i = prefix[i] - min(prefix[0..i-1])
        maxSum = Math.max(maxSum, prefix[i] - minPrefix);
        minPrefix = Math.min(minPrefix, prefix[i]);
    }
    
    return maxSum;
}

// Test
console.log(maxSubArrayPrefixSum([-2, 1, -3, 4, -1, 2, 1, -5, 4])); // Output: 6
```

---

### Key Takeaways

| Technique | Best For | Time Complexity | Space Complexity |
|-----------|----------|-----------------|------------------|
| **Prefix Sum** | Range queries, subarray sums | O(n) preprocess, O(1) query | O(n) |
| **Kadane's Algorithm** | Maximum/Minimum subarray problems | O(n) | O(1) |

**When to use:**
- Use **Prefix Sum** when you need to answer multiple range sum queries or find subarrays with specific sum properties.
- Use **Kadane's Algorithm** when finding the maximum (or minimum) sum of a contiguous subarray in a single pass.

Both techniques are fundamental for solving array-based problems efficiently and are frequently asked in FAANG interviews.
