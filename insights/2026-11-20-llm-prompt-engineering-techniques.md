# 📌 LLM prompt engineering techniques
*November 20, 2026 · Daily Dev Insight*

## 🧠 Overview

Prompt engineering has evolved from an experimental art into a crucial engineering discipline. As LLMs become core infrastructure in production systems, the way we craft prompts directly impacts reliability, cost, and user experience. Unlike traditional APIs where you define strict schemas, LLMs respond to nuanced instructions—making prompt design as critical as database query optimization was in the 2010s.

The best prompt engineers think like compilers: they translate high-level intent into precise instructions that guide the model toward consistent, high-quality outputs. This involves understanding few-shot learning, chain-of-thought reasoning, role-based prompting, and output formatting constraints. The difference between a novice and expert prompt can mean 40% accuracy versus 95% accuracy on the same task.

What separates production-grade prompting from playground experiments is systematic testing, versioning, and iteration. Your prompts should live in version control, have unit tests, and be treated as first-class code artifacts. The techniques we'll explore today aren't just tricks—they're fundamental patterns for building reliable LLM-powered features.

## 💡 Key Concepts

- **Few-shot learning**: Provide 2-5 examples of the desired input-output pattern. This dramatically improves consistency and teaches the model format, style, and edge case handling without fine-tuning.

- **Chain-of-thought (CoT)**: Ask the model to "think step by step" or "explain your reasoning." This unlocks better performance on complex tasks by forcing intermediate reasoning steps before the final answer.

- **Role and context framing**: Start with explicit role assignment ("You are an expert data scientist...") and provide relevant context. This primes the model's behavior and reduces hallucinations.

- **Output formatting constraints**: Use XML tags, JSON schemas, or structured delimiters to ensure parseable outputs. Never rely on markdown formatting alone in production systems.

- **Temperature and token management**: Lower temperatures (0.1-0.3) for deterministic tasks; higher (0.7-0.9) for creative work. Always set `max_tokens` to prevent runaway costs and implement streaming for better UX.

## 🐍 Python Example

```python
import anthropic
import json

def extract_structured_data(user_review: str) -> dict:
    """
    Extract sentiment and key points from user reviews using
    few-shot prompting and structured output formatting.
    """
    client = anthropic.Anthropic()
    
    # Few-shot examples teach the model the pattern
    prompt = f"""You are a sentiment analysis expert. Extract sentiment and key points from reviews.

Examples:
<example>
Review: "Great product but shipping took forever."
Output: {{"sentiment": "mixed", "key_points": ["positive product quality", "slow shipping"], "confidence": 0.9}}
</example>

<example>
Review: "Absolutely love it! Worth every penny."
Output: {{"sentiment": "positive", "key_points": ["high satisfaction", "good value"], "confidence": 0.95}}
</example>

Now analyze this review:
<review>
{user_review}
</review>

Provide output as valid JSON only, no markdown formatting."""

    response = client.messages.create(
        model="claude-3-5-sonnet-20261022",
        max_tokens=256,
        temperature=0.2,  # Low temp for consistent structured output
        messages=[{"role": "user", "content": prompt}]
    )
    
    # Parse and validate the structured output
    result = json.loads(response.content[0].text)
    
    # Validate expected schema
    assert "sentiment" in result and "key_points" in result
    return result

# Usage
review = "The UI is intuitive but crashes on Android 12."
analysis = extract_structured_data(review)
print(f"Sentiment: {analysis['sentiment']}")
print(f"Points: {', '.join(analysis['key_points'])}")
```

## 🟨 JavaScript Example

```javascript
import Anthropic from '@anthropic-ai/sdk';

/**
 * Generate test cases using chain-of-thought reasoning
 * to ensure comprehensive edge case coverage
 */
async function generateTestCases(functionDescription, functionSignature) {
  const anthropic = new Anthropic({
    apiKey: process.env.ANTHROPIC_API_KEY,
  });

  // CoT prompt guides the model through systematic thinking
  const prompt = `You are a senior QA engineer. Generate test cases for this function.

Function: ${functionSignature}
Description: ${functionDescription}

Think step by step:
1. First, identify the input types and edge cases
2. Consider boundary conditions and error states
3. Think about performance and security implications
4. Generate test cases in order: happy path, edge cases, errors

Output format:
<test_cases>
<test name="descriptive_name" input="..." expected="..." />
</test_cases>

Begin your analysis:`;

  const message = await anthropic.messages.create({
    model: 'claude-3-5-sonnet-20261022',
    max_tokens: 1024,
    temperature: 0.3, // Balanced for systematic but creative thinking
    messages: [{ role: 'user', content: prompt }],
  });

  const responseText = message.content[0].text;
  
  // Extract structured test cases using regex
  const testPattern = /<test name="([^"]+)" input="([^"]+)" expected="([^"]+)"/g;
  const tests = [];
  let match;
  
  while ((match = testPattern.exec(responseText)) !== null) {
    tests.push({
      name: match[1],
      input: match[2],
      expected: match[3],
    });
  }
  
  return tests;
}

// Usage
const tests = await generateTestCases(
  'Validates email addresses',
  'function isValidEmail(email: string): boolean'
);

tests.forEach(t => console.log(`Test: ${t.name}\n  Input: ${t.input}\n  Expected: ${t.expected}\n`));
```

## ⚖️ When To Use / When To Avoid

**Use LLM prompt engineering when:**
- You need flexible, natural language understanding (sentiment analysis, classification, extraction)
- The problem space is too broad for traditional regex/rules (entity recognition, intent detection)
- You want rapid prototyping without training custom models
- You have variable input formats that need normalization

**Avoid or supplement with traditional code when:**
- You need deterministic outputs for critical business logic (use LLMs for suggestions, not decisions)
- Latency requirements are under 100ms (consider caching layers or smaller models)
- The task is simple pattern matching (regex is faster and cheaper)
- You're processing sensitive data that can't leave your infrastructure (consider self-hosted models)

## 📚 Further Reading

- [Anthropic Prompt Engineering Guide](https://docs.anthropic.com/claude/docs/prompt-engineering) – Comprehensive best practices from the Claude team
- [OpenAI Prompt Engineering Documentation](https://platform.openai.com/docs/guides/prompt-engineering) – Techniques and examples for GPT models
- [Few-Shot Learning Paper (Brown et al.)](https://arxiv.org/abs/2005.14165) – Original GPT-3 paper demonstrating few-shot capabilities
- [Chain-of-Thought Prompting (Wei et al.)](https://arxiv.org/abs/2201.11903) – Research paper on CoT reasoning improvements
- [LangChain Prompt Templates](https://python.langchain.com/docs/modules/model_io/prompts/) – Production-ready prompt management patterns

---
*Auto-generated by [Daily Dev Insights Bot](https://github.com) · Powered by Claude AI*