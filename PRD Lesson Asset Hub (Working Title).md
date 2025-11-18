# PRD: Lesson Asset Hub (Working Title)

### 1. Introduction & Problem Statement

**Problem:** Our EdTech company's core assets (lesson plans, videos, images, source files) are stored in Google Drive. While Google Drive is excellent for storage, it's not a library. Assets are difficult to find, discover, or reuse. Content creators waste significant time searching for materials, or worse, recreating assets that already exist. There is no standardized metadata (e.g., school level, subject), making it impossible to browse or manage our content catalog effectively.

**Vision:** The **Lesson Asset Hub** will be a web application wrapper for our Google Drive. It will automatically crawl, analyze, and tag all lesson materials, making every single asset—from a Figma design to a Grade 3 math video—instantly searchable and discoverable through a single, intelligent interface.

### 2. Goals & Objectives

* **Product Goal 1 (Crawl & Tag):** Automatically crawl specified Google Drive folders, identify all files, and assign a rich set of metadata based on folder structure and, eventually, file content.
* **Product Goal 2 (Search & Discover):** Provide a powerful, faceted search interface that allows users (e.g., curriculum designers) to find assets in seconds using filters like `Subject`, `Grade Level`, `File Type`, or `Lesson`.
* **Business Goal:** Decrease content production time by 25% by increasing asset reuse and reducing time spent searching.
* **Business Goal:** Establish a "single source of truth" for all lesson-related digital assets.

### 3. User Personas

* **Priya, Curriculum Designer:** Needs to build a new lesson for "Grade 5, Science, The Solar System." She needs to find existing videos, illustrations, and text documents related to that topic. She currently has to ask colleagues in a chatroom where to find things.
* **David, Video Editor (Vendor):** Needs to upload a final MP4 for a specific lesson. He needs to know *where* to put it so Priya can find it, and he needs to access the source Figma/Photoshop files.
* **Sarah, Team Lead:** Needs to conduct an audit of all "Grade 2 English" materials to check for brand consistency. She needs a high-level view of all assets for that category.

### 4. Core Features (Epics & User Stories)

This is the functional breakdown of your two main goals.

#### Epic 1: Google Drive Integration & Crawling
* **Story:** As an Admin, I need to securely connect the application to our company's Google Drive using OAuth, so the app can read our file structure.
* **Story:** As the System, I need to perform a one-time initial crawl of all specified folders and files to build the initial index.
* **Story:** As the System, I need to use Google Drive's API (e.g., webhooks) to listen for real-time changes (file added, moved, deleted) so the index is always up-to-date.

#### Epic 2: Automated Metadata Engine
* **Story:** As the System, I need to parse the folder path of a file to automatically assign metadata.
    * *Example:* A file at `/Subjects/English/Grade 4/Lesson - 'Nature and Environment'/Part - 'A Walk in the Garden'/video.mp4`
    * *Should be tagged as:*
        * `Subject`: "English"
        * `Grade Level`: "Grade 4"
        * `Lesson`: "Nature and Environment"
        * `Lesson Part`: "A Walk in the Garden"
        * `File Type`: "video"
* **Story:** As the System, I need to identify the file type (e.g., `video`, `image`, `document`, `design_source`) for filtering.
* **Story:** As a User, I need the system to read basic metadata from the file itself (e.g., `Year Created`, `Author`).

#### Epic 3: The Search Interface (The Wrapper)
* **Story:** As Priya, I need a single search bar where I can type "space" and see all assets related to space.
* **Story:** As Priya, I need to be able to filter search results by:
    * `File Type` (Video, Image, Document, Design File)
    * `Subject` (English, Math, Science)
    * `Grade Level` (K, 1, 2, ... 12)
    * `Lesson`
* **Story:** As Priya, when I click on a search result, I need to see a preview of the asset (e.g., a thumbnail for images/video, first page for text).
* **Story:** As Priya, from the search result, I need a direct link to open the file in Google Drive.

### 5. Non-Functional Requirements

* **Security:** The app **must** respect all existing Google Drive permissions. A user must *never* be able to see a file in the search app that they don't have permission to see in Google Drive.
* **Performance:** Search results must return in under 2 seconds.
* **Scalability:** The system must be able to handle an index of 1,000,000+ assets.

### 6. Success Metrics

* **Primary:** Time-to-find-asset (TTFA) (measured via user testing).
* **Secondary:** Daily Active Users (DAU) of the Hub.
* **Business:** % of new lessons created that reuse at least 3 existing assets.

### 7. Open Questions & Assumptions

* **Assumption:** Our Google Drive has a clean, consistent, and enforceable folder structure. (If not, this project is 10x harder, and Epic 2's primary story is invalid).
* **Question:** What level of file preview is needed for design files (Figma, PSD)? Is a link sufficient, or do we need to generate image thumbnails?
* **Question:** How will we handle non-structured content (e.g., files in the "Shared with me" root)? (Proposal: We only index specified "Library" folders).