# SpeakEZ Blog Writing Style Guide

Based on analysis of blog posts from August 2025 - January 2026.

## Core Principles

1. **No Em Dashes**: Never use em dashes (—). Use periods, commas, or restructure sentences.

2. **Narrative Expansion Over Bullets**:
   - Avoid bullet lists for main content
   - Expand ideas into full paragraphs with careful exposition
   - Use bullets sparingly (perhaps for technical lists, code features)
   - Prefer numbered lists for sequences or steps

3. **Historical Context and Evolution**:
   - Root technical decisions in history (e.g., "50 years of actor model")
   - Show the progression of ideas
   - Reference foundational papers and creators
   - Explain why now is the right moment

4. **Careful, Deliberate Exposition**:
   - Take time to explain concepts thoroughly
   - Build understanding progressively
   - Don't assume reader knowledge
   - Define terms before using them extensively

5. **Thematic Organization**:
   - Use clear section headings that tell a story
   - Headings should be descriptive, not cryptic
   - Structure guides the reader through the narrative arc

6. **Real Code Examples**:
   - Include substantive code blocks with explanation
   - Show before/after comparisons
   - Explain what the code does and why it matters
   - Use F# code that compiles and runs

7. **Architectural Diagrams**:
   - Mermaid diagrams for complex architectures
   - Show relationships between components
   - Visual representation complements text

8. **Cross-References**:
   - Link to related blog posts extensively
   - Format: [descriptive text](/blog/post-slug/)
   - Build an interconnected web of ideas
   - Help readers discover related content

9. **Technical Depth with Accessibility**:
   - Don't dumb down technical content
   - Explain complex ideas clearly
   - Assume intelligent reader, not expert reader
   - Bridge from familiar to unfamiliar

10. **Source Preservation**:
    - Always cite sources with proper links
    - Credit open source projects and creators
    - Acknowledge influences and inspirations
    - Link to official documentation

## Structural Patterns

### Opening
- Start with the problem or question
- Establish context and stakes
- Lead reader into the topic naturally

### Body
- Progressive disclosure of complexity
- Each section builds on previous
- Use transitions between sections
- Return to themes established in opening

### Code Examples
- Introduce with context
- Show the code
- Explain what it does
- Explain why it matters
- Connect to larger architecture

### Closing
- Synthesize main points
- Connect to broader vision
- Point to future directions
- Encourage engagement

## Voice and Tone

- Professional but not corporate
- Enthusiastic but not hyperbolic
- Confident but not arrogant
- Educational but not condescending
- Technical but not impenetrable

## Common Phrases and Patterns

- "This isn't merely..." / "This represents more than..."
- "The X model provides..."
- "Consider what happens when..."
- "The architecture enables..."
- "By combining X with Y, we achieve Z"
- "This convergence of..."
- Reference to "standing on shoulders of giants" (Fable, Hawaii, Glutinum, etc.)

## What to Avoid

- Em dashes (—)
- Excessive bullet points in main narrative
- Jargon without explanation
- Assumptions about reader knowledge
- Over-simplified analogies that condescend
- Hype and marketing speak
- Vague time references ("soon", "eventually")

## Example Analysis: "Actors Take Center Stage" (Sept 2025)

This post exemplifies the SpeakEZ style:

- **Historical grounding**: Opens with "50 years of actor model" context
- **Narrative progression**: Moves from Erlang origins through F# MailboxProcessor to CloudflareFS
- **No em dashes**: Uses periods and careful sentence construction
- **Real code examples**: Shows actual F# actor code with detailed explanation
- **Cross-references**: Links to related posts throughout
- **Thematic headings**: "Architecture as Destiny", "The Worker Loader: Dynamic Actors"
- **Technical depth**: Explains Worker Loader mechanism without oversimplifying

## Example Analysis: "Getting The Signal With BAREWire" (Dec 2025)

Another strong example:

- **Problem-first opening**: "Reactive programming has become essential..." establishes context
- **Careful exposition**: Explains subscription problem before presenting solution
- **Progressive complexity**: Builds from simple signals to distributed coordination
- **Code walkthrough**: Shows implementation with explanation of why each part matters
- **Architectural diagrams**: Mermaid diagrams complement text explanations
- **Cross-linking**: References multiple related posts to build interconnected narrative
