# UI - Design System

## Colour Palette

The interface employs a refined, minimal colour system inspired by European modernism:

- **Neutral foundation**: Slightly darker off-white background (#f4f4f4) with pure white cards (#ffffff) creates noticeable contrast whilst keeping the interface light. Borders use #dfdfdf for crisp delineation without heaviness.
- **Text hierarchy**: Primary text #1a1a1a (near-black), secondary #4a4a4a (rich grey), muted #7a7a7a for supporting information. This ensures accessible contrast against the darker canvas.
- **Accent colour**: Vibrant blue (#2563eb) for primary actions and interactive elements, providing visual focus without distraction.
- **Priority indicators**: Red (#cc0000) for high priority, blue (#0000cc) for medium, yellow (#cccc00) for low. These primary colours create immediate visual recognition.
- **Semantic colours**: Green (#059669) for success, orange (#d97706) for warning, red (#dc2626) for danger states.

#
## Grid and Layout Structure

The layout follows strict grid principles aligned with Swiss design methodology:

- **Header**: Fixed bar (73px height) spanning full width with 1px bottom border, containing title and action controls. Uses flex layout for even horizontal distribution.
- **Sidebar**: 280px fixed width with consistent padding (24px). Sections organised vertically with 32px gaps between sections, uppercase category titles with 0.05em letter spacing.
- **Main content area**: Flexible columns (To Do, In Progress, Review, Done) using CSS Grid. Each column has consistent width and maintains vertical scroll within the main container.
- **Spacing system**: Modular scale from 0.25rem to 3rem (8 increments). Every element uses these defined values for perfect alignment and rhythm.
- **Card system**: Consistent padding and rounded corners (6px) create visual cohesion. Cards stack vertically within columns with small gaps, allowing drag and drop interaction.

## Typography System

Helvetica-inspired, sans-serif foundation reflects classic Swiss grid design:

- **Font stack**: System fonts (-apple-system, Segoe UI, Helvetica Neue) ensure clarity and performance whilst maintaining European aesthetic sensibilities.
- **Modular scale**: 0.75rem (extra small) through 1.875rem (heading level 1). Each step represents intentional visual hierarchy.
- **Line heights**: 1.1 (tight for headings), 1.3 (normal body), 1.45 (relaxed for longer content). Creates breathing room whilst maintaining density.
- **Weight hierarchy**: 400 (regular), 500 (medium for buttons), 600 (semibold for subtitles), 700 (bold for primary headings).
- **Uppercase treatment**: Section titles in uppercase with letter spacing reinforce hierarchy and structure, echoing Swiss design conventions.

## Design Philosophy - Swiss Grid and European Modernism

This interface embodies principles established by Erik Spiekermann and the European design tradition:

- **Functional minimalism**: No decorative flourishes. Every visual element serves navigation or information purposes.
- **Strict alignment**: Grid-based layout ensures predictable, rational structure. Elements align vertically and horizontally with precision.
- **Generous whitespace**: Off-white backgrounds and padding create optical separation between functional areas. Information breathes.
- **Limited colour palette**: Neutral greys with single accent colour reduces cognitive load and guides user attention to actionable items.
- **Clear hierarchy**: Font sizes, weights, and spacing create unambiguous content priority.
- **Legibility first**: Helvetica-derived typefaces and high contrast text ensure international usability, reflecting the Swiss principle of universal communication.

## Application Surfaces & Micro-Patterns

- **Search Command Bar**: Prominent global search input paired with an advanced filter tray. Preset chips display current constraints (subject, grade range, metadata tags, owner, last-modified window). Results list surfaces match score, Drive path, and “Open in Drive” actions.
- **Crawl Workboard**: Four-column Kanban (To Do, In Progress, Review, Done). Cards include progress bars (grey track #c7c7cc with accent fill #2563eb), priority pills, asset counts, and metadata rows that align labels/values via fixed-width grids.
- **Queue Drill-down Cards**: Linked via “View queue” anchors. Each queue card shows file-level status table with status pills (Queued grey, Processing blue, Ready green), owner, and last-touch info for transparency.
- **File Type Messaging**: Asset types communicate using extensions (JPG, MP4, PSD, DOCX, etc.) for immediate recognition by production teams.