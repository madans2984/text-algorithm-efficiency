# Efficiency of Knuth-Morris-Pratt compared to Rabin-Karp over different text sizes
## By Shree Madan
### 1. Introduction
For my DSA final project, I decided to implement two different string-matching algorithms in go and compare their running times on texts of different sizes. I chose to explore the Knuth-Morris-Pratt(KMP) and Rabin-Karp(RK) algorithms and used text sizes of a word, a paragraph, a chapter, and a book. Although these aren't exact word counts, they demonstrate the difference in scale between each test. An important piece of background information is about naive search algorithms. A naive search algorithm would take a pattern string and compare it against every character in a text string which is a simple but inefficient method of string matching. Both KMP and RK attempt to improve on this in different methods but both use preprocessing to try and make the process faster.
### 2. Knuth-Morris-Pratt
KMP is relatively closer to naive searching. This algorithm uses a prefix table (derived from the pattern) to try and determine which characters don't need to be checked and therefore skipped. Once this pre-fix table is generated, the algorithm checks every character but depending on where the character mismatches we can use on pre-fix table to determine how many comparisons we can skip (shifting the pattern to the right).
#### 2.1 Generating Prefix Table
Generating the prefix table is a conceptually difficult step. The prefix table is also known as the LPS (Longest Proper Prefix that is also a Suffix). The Proper Prefix/Suffix of a string are the strings that are prefixes or suffixes without including the entire string. For example "ABC" would be a prefix to "ABCD" but "ACBD" cannot be a prefix to itself. The vice-versa is true for suffixes where the strings won't include the first character. The values in the prefix table correspond with the length of the longest Proper Prefix which is also a Proper Suffix in the same substring. 
We create our prefix table to be the same length as our pattern. It's much easier to explain this with an example so let's walk through how we would generate a prefix table for the string "ABABA". 
- The first character will always have a 0 in the table as there are no Proper Prefixes or Suffixes for a string of length 1. 
- For the second character, our possible prefix is "A" and our possible suffix is "B" since these don't match we leave the 0 in that column. 
- For the third character, the possible prefixes are "A" and "AB" and the possible suffixes are "BA" and "A". Since "A" is a prefix and suffix, we add its length (1) as the value in the table for that character.
- For the fourth character, the possible prefixes are "A" "AB", "ABA" and the possible suffixes are "B", "AB", and "BAB". Since "AB" is a prefix and suffix, we add its length (2) as the value in the table for that character.
- For the fifth character, the possible prefixes are "A" "AB", "ABA", "ABAB" and the possible suffixes are "A", "BA", "ABA", "BABA". Since "ABA" is a prefix and suffix, we add its length (3) as the value in the table for that character.

| index | 0 | 1 | 2 | 3 | 4 |
| --- | --- | --- | --- | --- | --- |
| character | A | B | A | B | A |
| value | 0 | 0 | 1 | 2 | 3 |

Below is the code implementation for this process.

    func preKMP(pattern string) []int {
        // create an empty array that's the size of the pattern length to store prefix values
        // LPS stands for Longest Proper Prefix that is also a Suffix
        LPS := make([]int, len(pattern) + 1)
        j := -1 // -1 represents when none of the prefixes and suffixes match
        LPS[0] = -1
    
        // loop runs for every letter in the pattern
        for i := 0; i < len(pattern) - 1; i++ {
            // if j is greater than -1, a string is being stored as a pattern so when the next character 
            // doesn't match it needs to reset. This keeps happening until j is either -1 OR when the 
            // characters match so if none of the characters matches, j will keep resetting until it hits -1
            for j > -1 && pattern[i] != pattern[j] {
                j = LPS[j]
            }
    
            // if j hits -1 then it needs to start the next character match so we increment
            j++
    
            // if the characters match, we set the table to equal the length of the longest 
            // identical prefix and suffix of the last match. If not, we set it to what 
            // the current match is.
            if pattern[i] == pattern[j] {
                LPS[i] = LPS[j]
            } else {
                LPS[i] = j
            }
        }
        return LPS
    }

#### 2.2 Implementing the Algorithm
To make sure, the text and pattern fit the use case (neither are empty and the pattern isn't longer than the text), we have a few built-in checks.

    n, m := len(text), len(pattern)

    // empty string for pattern or text, panic!
    if m == 0 || n == 0 {
        panic("Your text or pattern is empty!")
    }

    // pattern is longer than text, panic!
    if n < m {
        panic("Your pattern is longer than your text!")
    }

To use our prefix table, we need to write a function that understands how to use the table to parse the text. The efficacy of using LPS in the searching function is that we have a few matches and then a mismatch, we have an idea of how many indices we can skip in our comparisons, reducing the total number of comparisons needed making it more efficient than a naive searching algorithm.

    next := preKMP(pattern) // creates prefix table
    i := 0 // number of characters matched so far
    j := 0 // index of where we are in the text
    
    // create an array to store start indexes of patterns in text
    var res []int
    
    // the loop will run through each index of the text incrementing based on the pre-fix table
    for j < n {
        // this loop goes through the pattern incrementally until there's a character match with the text
        for i > -1 && pattern[i] != text[j] {
            i = next[i]
        }
        // once there's a match, we'll need to increment both indexes
        i++
        j++
        // only when i is greater than or equal to the length of the pattern, 
        // we know that the entire pattern matches and can add the index to our result.
        if i >= m {
            res = append(res, j-i)
            i = next[i]
        }
    }
    return res 
    
An example of what this would return is below:

    txt := "ABBAABADABABBAA"
    pattern := "ABBA"
    fmt.Println(KMP(txt, pattern))
    Return: [0 10]
### 3. Rabin-Karp
RK attempts to use hashing as a way of streamlining the search process. Similar to KMP, we're trying to reduce the number of comparisons we need to make to find the pattern in the text. The algorithm started by computing a hash value for the pattern string. We then take the text and have a sliding window of the pattern length. This window starts at the first index and computes each hash value of each substring with the same length as the pattern. This results in a table where we have the index of the first character and the hash value. By comparing the hash values in this table to the original value computed for the pattern, we can say that every index which has the same hash value is the pattern. The reason this works is due to the collision-resistant nature of hash values which gives us a very high likelihood that if the hash values are the same, the string will be the same. The two main helper functions for this algorithm are the hash function to compute the values and a match function to parse the table and compare it to the pattern value. These are then used in the implementation below. Similar to KMP, we start by ensuring the text and pattern meet the use-case requirements. We then create the hash values for the pattern and all the windows (which are appended to a table). After this, we're able to match the values and output the indices where the pattern matches the text.

    func RK(txt string, patterns string) []int {
        n, m := len(txt), len(patterns)
        
        //empty string for pattern or text, panic!
        if m == 0 || n == 0 {
            panic("Your text or pattern is empty!")
        }
    
        //pattern is longer than text, panic!
        if n < m {
            panic("Your pattern is longer than your text!")
        }
    
        // creating an empty hash table for the text
        h := make(map[int]uint32)
        // determining the hash value of the pattern
        hp := hash(patterns)
        // starting with 0 slides a window the same size as the pattern over the entire text
        // computes the hash and stores it with the first index of the window
        for i := 0; i < n - m + 1; i++ {
            h[i] = hash(txt[i:i+m])
        }
    
        // searches through text hash table with the hash value of pattern
        res := match(h, hp)
        // sort result
        BubbleSort(res)
    
        return res
    }
    
An example of what this would return is below:

    txt := "ABBAABADABABBAA"
    pattern := "ABBA"
    fmt.Println(RK(txt, pattern))
    Return: [0 10]
### 4. Use Cases and Time Comparison
There are two kinds of use cases for matching algorithms, single pattern and multiple pattern matching. I've only implemented single pattern matching for each of them which is not the best use for both. KMP is a superior algorithm to RK when it comes to single pattern matching however RK is better for multiple pattern matching as it's able to quickly compare whether the hash value of a substring exists in the hash values of the patterns we are looking for. I implemented a simple timing function that takes the two algorithms, ensures they get the same result, and then outputs the average of 10 iterations of the search. For small texts such as:

    txt0 := "cocacola"
    pattern0 := "co"
    fmt.Println(compare(txt0, pattern0))
    
returns as [0, 0] a majority of the time. However, as we increase the word count, the difference becomes more and more apparent. With a sample of 640 words, the time returns at around [440, 10] for RK and KMP respectively which starts the show how much quicker the KMP algorithm is. As shown in the table below as the word count increases, the difference becomes more dramatic. These are all in microseconds so in terms of human processing it doesn't seem that different but KMP in the long run will be significantly more efficient than RK for single pattern matching. 

| word count | 1 | 640 | 2189 | 9755 |
| --- | --- | --- | --- | --- |
| RK | 0 | 440 | 920 | 3950 |
| KMP | 0 | 10 | 30 | 130 |

This is also supported by the time complexities of the two algorithms. For the worst case of KMP, we can break it down into the two parts of the function. We can create the LPS table in O(m) and the actual searching aspect of the function can be done in O(n) making the complexity of the entire algorithm O(m+n). A common misconception is that we have worst-case time on O(mn) due to the fact that we may need to do multiple iterations of indexing the pattern for each index of the text. However, we never decrement the index which is parsing the text giving us a complexity of n. We do decrement the pattern index but each time we decrement it we can only decrement the same amount of steps as we had matches. So in the end to have n decrementing steps, we would need n matching steps making our complexity n+n = 2n or O(n). In comparison the worst case complexity of RK is O(mn). The complexity of this algorithm is largely affected by the effectiveness of the hash function used but when searching for all instances of a pattern in a text (like I do in the tests) it's much easier to need to O(mn) since every index needs to be checked against the pattern. This is actually identical to a naive searching algorithm but it's only like this when we are only matching against one pattern. When we are checking against multiple patterns, RK is much better as it only needs to compute the hash of the text windows once. 
