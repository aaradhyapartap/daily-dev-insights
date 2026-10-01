# 📌 LLM prompt engineering techniques
*October 01, 2026 · Daily Dev Insight*

## 🧠 Overview

Prompt engineering has evolved from a dark art into a proper discipline. As LLMs become infrastructure rather than novelty, the ability to craft effective prompts is as fundamental as writing good SQL queries or designing clean APIs. The difference between a mediocre and excellent prompt can mean the gap between 40% accuracy and 95% accuracy, or between a system that hallucinates constantly and one that reliably delivers value.

The core insight is this: LLMs are simulators, not databases. They don't "know" facts—they predict likely continuations based on patterns. Your prompt isn't just a question; it's context-setting, role-assignment, and constraint-definition all rolled into one. The best prompts leverage few-shot learning, chain-of-thought reasoning, and clear output formatting to guide the model toward consistent, useful responses.

Modern prompt engineering borrows from compiler design (be explicit about syntax), UX design (reduce cognitive load), and traditional ML (validation and iteration). As we integrate LLMs deeper into production systems, treating prompts as versioned, tested artifacts is no longer optional—it's essential.

## 💡 Key Concepts

- **Few-shot learning**: Provide 2-5 examples of input/output pairs before your actual query. The model pattern-matches far better than with zero-shot instructions alone.

- **Chain-of-thought (CoT)**: Add "Let's think step by step" or show reasoning in examples. This dramatically improves performance on logic, math, and multi-step tasks by forcing intermediate reasoning.

- **System/User/Assistant framing**: Structure prompts with clear role separation. System messages set behavior, user messages provide input, assistant messages show desired format.

- **Output constraints**: Specify exact formats (JSON, bullet points, code only). Use delimiters like XML tags or triple-backticks to clearly separate different sections of your prompt.

- **Temperature and token limits**: Engineering isn't just the prompt text—tune sampling parameters. Low temperature (0.1-0.3) for deterministic tasks, higher (0.7-0.9) for creative work.

## �🐍 Python Example

```python
import anthropic
import json

def extract_structured_data(text: str) -> dict:
    """
    Extract structured product info using few-shot prompting
    and explicit JSON output constraints.
    """
    client = anthropic.Anthropic()
    
    # Few-shot examples embedded in the prompt
    prompt = f"""Extract product information into JSON format.

Examples:
Input: "The UltraBook Pro costs $1,299 and has 16GB RAM"
Output: {{"name": "UltraBook Pro", "price": 1299, "specs": {{"ram": "16GB"}}}}

Input: "Buy the SpeedRunner shoes for $89.99, available in red"
Output: {{"name": "SpeedRunner shoes", "price": 89.99, "specs": {{"color": "red"}}}}

Now extract from this text:
Input: "{text}"
Output:"""

    message = client.messages.create(
        model="claude-3-5-sonnet-20241022",
        max_tokens=256,
        temperature=0.1,  # Low temp for consistency
        system="You are a precise data extraction assistant. Only output valid JSON.",
        messages=[{"role": "user", "content": prompt}]
    )
    
    # Parse and validate the response
    try:
        result = json.loads(message.content[0].text)
        return result
    except json.JSONDecodeError:
        return {"error": "Failed to parse JSON", "raw": message.content[0].text}

# Example usage
product_text = "The PowerDrill 3000 is only $149 with a 2-year warranty"
data = extract_structured_data(product_text)
print(json.dumps(data, indent=2))
```

## 🟨 JavaScript Example

```javascript
import Anthropic from '@anthropic-ai/sdk';

/**
 * Code review assistant using chain-of-thought prompting
 * to provide structured, reasoned feedback
 */
async function reviewCode(code, language = 'javascript') {
  const anthropic = new Anthropic({
    apiKey: process.env.ANTHROPIC_API_KEY
  });

  // Chain-of-thought prompt structure
  const systemPrompt = `You are an expert code reviewer. 
For each review:
1. First, identify the code's purpose
2. Check for bugs or edge cases
3. Evaluate readability and style
4. Suggest specific improvements

Format as: PURPOSE | ISSUES | SUGGESTIONS`;

  const userPrompt = `Review this ${language} code:

\`\`\`${language}
${code}
\`\`\`

Let's think through this step by step.`;

  const message = await anthropic.messages.create({
    model: 'claude-3-5-sonnet-20241022',
    max_tokens: 1024,
    temperature: 0.3,
    system: systemPrompt,
    messages: [
      { role: 'user', content: userPrompt }
    ]
  });

  return message.content[0].text;
}

// Example usage
const sampleCode = `
function calc(a, b) {
  return a + b / 2;
}`;

reviewCode(sampleCode)
  .then(review => console.log(review))
  .catch(err => console.error('Review failed:', err));
```

## ⚖️ When To Use / When To Avoid

**Use prompt engineering when:**
- You need consistent, structured outputs (data extraction, classification)
- The task benefits from examples (few-shot learning excels here)
- You're working with well-defined domains and can provide clear constraints
- Cost and latency are acceptable for the value gained

**Avoid or supplement with fine-tuning when:**
- You need millisecond response times (prompts add latency)
- You have 10,000+ examples and need maximum accuracy
- The task requires deep domain knowledge not in the training data
- You're doing simple keyword matching (use traditional search)

## 📚 Further Reading

- [Anthropic's Prompt Engineering Guide](https://docs.anthropic.com/en/docs/prompt-engineering) - Comprehensive techniques with interactive examples
- [OpenAI Prompt Engineering Best Practices](https://platform.openai.com/docs/guides/prompt-engineering) - Strategies from the GPT team
- [Few-Shot Learning Research Paper (Brown et al.)](https://arxiv.org/abs/2005.14165) - The academic foundation for few-shot prompting
- [Prompt Engineering Guide (GitHub)](https://github.com/dair-ai/Prompt-Engineering-Guide) - Community-maintained resource with 40k+ stars
- [Chain-of-Thought Prompting Paper](https://arxiv.org/abs/2201.11903) - Original research on CoT reasoning

---
*Auto-generated by [Daily Dev Insights Bot](https://github.com) · Powered by Claude AI*