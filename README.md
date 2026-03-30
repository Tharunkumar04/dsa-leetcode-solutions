# DSA LeetCode Solutions – FAANG Preparation

---

##  Overview

This repository contains structured LeetCode DSA solutions focused on:

- Problem-solving patterns  
- Optimized approaches  
- FAANG-level interview preparation  

---

## Core Skills Demonstrated

- Algorithm Design  
- Data Structures Mastery  
- Complexity Analysis (Big-O)  
- Pattern Recognition  
- Optimization Techniques  
- Edge Case Handling  

---

## Folder Structure

```text
dsa-leetcode-solutions/
│
├── Arrays               # Prefix Sum, Kadane's Algorithm
├── Strings              # Hashing, Pattern Matching
├── LinkedList           # Fast & Slow Pointers
├── Stack                # Monotonic Stack
├── Queue                # Sliding Window
├── Trees                # DFS, BFS, Traversals
├── Graphs               # DFS, BFS, Topological Sort
├── DynamicProgramming   # Memoization, Tabulation
├── Recursion            # Divide & Conquer
├── Backtracking         # Combinations, Permutations
└── SlidingWindow        # Fixed & Variable Window
```

---

## Progress Tracker

| Topic                | Solved |
|---------------------|--------|
| Arrays              | 0      |
| Strings             | 0      |
| LinkedList          | 0      |
| Stack               | 0      |
| Queue               | 0      |
| Trees               | 0      |
| Graphs              | 0      |
| DynamicProgramming  | 0      |

---

## Problem Solving Approach

1. Understand the problem  
2. Identify pattern  
3. Start with brute force  
4. Optimize step-by-step  
5. Analyze complexity  
6. Test edge cases  

---

## Example

```javascript
function twoSum(nums, target) {
    const map = new Map();

    for (let i = 0; i < nums.length; i++) {
        const complement = target - nums[i];

        if (map.has(complement)) {
            return [map.get(complement), i];
        }

        map.set(nums[i], i);
    }
}
```

---

## Tech Stack

- JavaScript
- Python 

---

## Goal

- Solve 300+ problems
- Master DSA patterns
- Crack FAANG interviews

---

##  Connect

- GitHub: [https://github.com/Tharunkumar04](https://github.com/Tharunkumar04)
- LinkedIn: [www.linkedin.com/in/kollutharunkumar](www.linkedin.com/in/kollutharunkumar)

---

## Note

Consistency + Pattern Recognition = Success in Coding Interviews
