# 📌 LLM prompt engineering techniques
*May 04, 2026 · Daily Dev Insight*

## 🧠 Overview

Prompt engineering has evolved from simple trial-and-error to a sophisticated discipline that can make or break your LLM integration. After two years of production experience with GPT-4, Claude, and other models, we've learned that the difference between a mediocre AI feature and a game-changing one often comes down to how you craft your prompts.

The key insight most developers miss is that LLMs are fundamentally pattern-matching engines trained on human communication. This means your prompts need to be structured like clear, professional instructions you'd give to a talented but literal-minded intern. The more context, examples, and constraints you provide upfront, the more consistent and useful your outputs become. Think of it as API design for natural language—garbage in, garbage out still applies.

## 💡 Key Concepts

• **Chain-of-thought prompting**: Ask the model to "think step by step" or show its reasoning process to dramatically improve accuracy on complex tasks
• **Few-shot learning**: Include 2-3 examples of the desired input/output format in your prompt template rather than hoping the model will guess correctly
• **System messages vs user prompts**: Use system messages for behavioral instructions and constraints, user messages for the actual task content
• **Temperature and token limits**: Lower temperatures (0.1-0.3) for consistent structured outputs, higher (0.7-0.9) for creative tasks
• **Prompt versioning**: Treat prompts like code—version them, A/B test changes, and maintain a prompt library with performance metrics

## 🐍 Python Example

```python
import openai
from typing import List, Dict
import json

class CodeReviewAgent:
    def __init__(self, api_key: str):
        self.client = openai.OpenAI(api_key=api_key)
        
        # System prompt with clear role and constraints
        self.system_prompt = """You are a senior software engineer conducting code reviews.
        Focus on: security vulnerabilities, performance issues, maintainability, and best practices.
        Always provide specific line references and actionable suggestions.
        Format your response as valid JSON with 'issues' and 'suggestions' arrays."""
    
    def review_code(self, code: str, language: str) -> Dict:
        # Few-shot example to establish format
        user_prompt = f"""
        Please review this {language} code:
        
        ```{language}
        {code}
        ```
        
        Example response format:
        {{
            "issues": [
                {{"line": 5, "severity": "high", "type": "security", "description": "SQL injection vulnerability"}}
            ],
            "suggestions": [
                {{"description": "Consider using parameterized queries", "impact": "Prevents SQL injection attacks"}}
            ],
            "overall_rating": "needs_work"
        }}
        """
        
        response = self.client.chat.completions.create(
            model="gpt-4",
            messages=[
                {"role": "system", "content": self.system_prompt},
                {"role": "user", "content": user_prompt}
            ],
            temperature=0.2,  # Low temperature for consistent structured output
            max_tokens=1000
        )
        
        try:
            return json.loads(response.choices[0].message.content)
        except json.JSONDecodeError:
            return {"error": "Failed to parse LLM response as JSON"}

# Usage example
reviewer = CodeReviewAgent("your-api-key")
result = reviewer.review_code("def login(user, pwd): query = f'SELECT * FROM users WHERE name={user}'", "python")
```

## 🟨 JavaScript Example

```javascript
class DocumentationGenerator {
  constructor(apiKey) {
    this.apiKey = apiKey;
    
    // Chain-of-thought prompt for complex reasoning
    this.systemPrompt = `You are a technical writer creating API documentation.
    Think through this step by step:
    1. Analyze the function signature and parameters
    2. Infer the purpose from the code logic
    3. Identify potential edge cases or errors
    4. Write clear, concise documentation with examples
    
    Always include parameter types, return values, and usage examples.`;
  }
  
  async generateDocs(functionCode, functionName) {
    const userPrompt = `
    Generate documentation for this function:
    
    \`\`\`javascript
    ${functionCode}
    \`\`\`
    
    Follow this format:
    ## ${functionName}
    
    **Description**: [Brief description]
    
    **Parameters**:
    - param1 (type): description
    
    **Returns**: type - description
    
    **Example**:
    \`\`\`javascript
    // usage example here
    \`\`\`
    
    **Notes**: Any important caveats or edge cases
    `;
    
    try {
      const response = await fetch('https://api.openai.com/v1/chat/completions', {
        method: 'POST',
        headers: {
          'Authorization': `Bearer ${this.apiKey}`,
          'Content-Type': 'application/json'
        },
        body: JSON.stringify({
          model: 'gpt-4',
          messages: [
            { role: 'system', content: this.systemPrompt },
            { role: 'user', content: userPrompt }
          ],
          temperature: 0.3,
          max_tokens: 800
        })
      });
      
      const data = await response.json();
      return data.choices[0].message.content;
      
    } catch (error) {
      console.error('Documentation generation failed:', error);
      return `# ${functionName}\n\n*Documentation generation failed*`;
    }
  }
}

// Usage
const docGen = new DocumentationGenerator('your-api-key');
const docs = await docGen.generateDocs(
  'function debounce(func, delay) { let timeoutId; return (...args) => { clearTimeout(timeoutId); timeoutId = setTimeout(() => func.apply(this, args), delay); }; }',
  'debounce'
);
```

## ⚖️ When To Use / When To Avoid

**Use LLM prompt engineering when:**
- Processing unstructured text data (emails, documents, user feedback)
- Generating content that needs creativity within constraints
- Complex reasoning tasks that benefit from chain-of-thought
- Rapid prototyping of text-based features

**Avoid when:**
- You need deterministic, mathematically precise outputs
- Real-time performance is critical (sub-100ms responses)
- Processing highly sensitive data without proper safeguards
- Simple rule-based logic would suffice (don't use a sledgehammer for thumbtacks)

## 📚 Further Reading

• [OpenAI Prompt Engineering Guide](https://platform.openai.com/docs/guides/prompt-engineering) - Official best practices and techniques
• [Anthropic's Constitutional AI Paper](https://arxiv.org/abs/2212.08073) - Deep dive into training helpful, harmless, and honest AI systems  
• [LangChain Prompt Templates Documentation](https://python.langchain.com/docs/modules/model_io/prompts/) - Practical tools for prompt management and versioning
• [Google's PaLM Prompt Engineering Guidelines](https://developers.generativeai.google/guide/prompt_best_practices) - Model-agnostic techniques that work across providers
• [Papers With Code: Prompt Engineering](https://paperswithcode.com/task/prompt-engineering) - Latest research and benchmarks in prompt optimization

---
*Auto-generated by [Daily Dev Insights Bot](https://github.com) · Powered by Claude AI*