## Technical Documentation

### Document Layout Analysis (DLA)

The Document Layout Analysis (DLA) component is responsible for identifying and classifying the structural elements within a PDF document. This process involves determining the location and type of various blocks such as text, tables, figures, and headers.

**1. Initial Layout Detection with `surya.layout`:**

The system utilizes the `surya.layout` library as the primary engine for performing initial layout detection. `surya.layout` is a powerful tool that analyzes the visual and textual information on each page to identify potential layout regions. The `surya.layout.LayoutPredictor` returns a list of `LayoutResult` objects, one for each processed page image. Each `LayoutResult` contains `image_bbox` (the bounding box of the page image segment that Surya analyzed, e.g., `[x0, y0, x1, y1]`) and a list of `LayoutBox` objects. Each `LayoutBox` represents a detected structural element and includes:
    *   `label`: A string indicating the predicted block type (e.g., 'text', 'table', 'figure').
    *   `polygon`: A list of `[x, y]` coordinate pairs defining the vertices of the block's bounding polygon, relative to the `image_bbox`.
    *   `position`: An integer indicating the reading order of the block on the page as determined by Surya.
    *   `top_k`: A dictionary mapping potential labels to their confidence scores (e.g., `{'text': 0.92, 'list': 0.05}`).

**2. `LayoutBuilder` - Processing and Integration:**

The `marker.builders.layout.LayoutBuilder` class orchestrates the layout analysis process. Its main responsibilities include:
    - Taking a `Document` object and a `PdfProvider` as input.
    - Invoking the `surya.layout` model to get layout predictions for each page.
    - Handling a `force_layout_block` option, which allows treating every page as a specific block type, bypassing the `surya.layout` model.
    - Rescaling coordinates: The polygon coordinates from `surya.layout` (which are relative to the potentially cropped and resized image Surya processed) are converted to the PDF's native coordinate system. This is done using the `layout_block.polygon.rescale(layout_page_size, provider_page_size)` method. `layout_page_size` is the `[width, height]` of the image segment Surya processed (derived from `layout_result.image_bbox`), while `provider_page_size` is the `[width, height]` of the full PDF page (from `page.polygon.size`). The `rescale` method (defined in `marker.schema.polygon.PolygonBox`) applies a scaling factor (`new_dim / old_dim`) to each coordinate.
    - Adding the identified layout blocks (e.g., `TextBlock`, `TableBlock`) to the respective `PageGroup` objects within the `Document`. The `top_k` dictionary from Surya's `LayoutBox` is also stored in the created `Block` object, with string labels converted to `BlockTypes` enum members (e.g., `BlockTypes.Text`).

Key configuration options for `LayoutBuilder`:
    - `layout_batch_size`: Specifies the batch size for the layout model. Defaults to an optimal size based on the device (e.g., 12 for CUDA, 6 for CPU).
    - `force_layout_block`: A string representing a block type (e.g., "Text"). If set, all pages will be treated as this block type.
    - `disable_tqdm`: A boolean to disable progress bars during layout detection.

**3. `LLMLayoutBuilder` - Refining Layout with LLM (Gemini):**

The `marker.builders.llm_layout.LLMLayoutBuilder` class extends `LayoutBuilder` to further refine the layout results using a Large Language Model (LLM), specifically Google's Gemini. This step aims to improve the accuracy of block classification, especially for ambiguous or complex regions.

Key functionalities:
    - It first performs the standard layout detection using the parent `LayoutBuilder`.
    - **Relabeling Blocks:** It then iterates through the detected blocks and applies relabeling logic based on certain criteria:
        - **Low Confidence:** If a block's classification confidence (from `surya.layout`) is below a `confidence_threshold` (default: 0.7), it triggers a relabeling attempt. The LLM is provided with the image of the block and the top-k predictions from `surya.layout`, and it chooses the most appropriate label from that list.
        - **Potential Complex Regions:** If a block is initially labeled as `Picture` or `Figure` but its height exceeds a `picture_height_threshold` (default: 80% of page height), it's considered a candidate for being a more complex region (like `ComplexRegion`, `Table`, or `Form`). The LLM is prompted to choose from these more specific labels.
    - **LLM Interaction:** It uses an `llm_service` (configured for Gemini) to send the block image and a specialized prompt to the LLM. The LLM's response, which includes the refined label, is then used to update the block type in the `Document`.
        - The placeholders in prompts are filled dynamically:
            *   `{potential_labels}`: For low-confidence relabeling, this is populated by iterating through the `block.top_k` items. For each `BlockTypes` enum (e.g., `BlockTypes.Text`) and its description (e.g., 'A block of text.'), it formats them as '- `Text` - A block of text.'. For complex region relabeling, a predefined list of types like `Figure`, `Picture`, `ComplexRegion`, `Table`, `Form` and their descriptions are used.
            *   `{top_k}`: This (only in `topk_relabelling_prompt`) is populated by formatting each item in `block.top_k` as '- `Text` - Confidence 0.921'.
        - Image for LLM: An image of the specific block is extracted using `image_block.get_image(document, highres=False, expansion=(expand, expand))`. The `expansion=(0.01, 0.01)` parameter slightly enlarges the block's bounding box by 1% of its width and height before cropping from the page image. This provides the LLM with a little extra visual context around the block.
        - The LLM service is expected to return a JSON response conforming to the `LayoutSchema` (defined in `marker.builders.llm_layout`), which requires an `image_description` (LLM's reasoning) and a `label` (the chosen block type string).
    - **Concurrency:** It uses a `ThreadPoolExecutor` to make concurrent requests to the LLM, controlled by `max_concurrency`.

Configuration options for `LLMLayoutBuilder`:
    - `google_api_key`: The API key for the Gemini model.
    - `confidence_threshold`: The confidence score below which a block is relabeled.
    - `picture_height_threshold`: The height ratio above which pictures/figures are considered for complex relabeling.
    - `model_name`: The specific Gemini model to use (e.g., "gemini-2.0-flash").
    - `max_concurrency`: Maximum concurrent requests to the LLM.
    - `topk_relabelling_prompt`: The prompt template used for low-confidence relabeling.
    - `complex_relabeling_prompt`: The prompt template used for complex region relabeling.

This multi-stage approach, combining traditional layout analysis with LLM-based refinement, allows the system to achieve a robust and accurate understanding of document structure.

### Cross-Page Processing

Effective document understanding often requires analyzing content that spans multiple pages. The system employs several mechanisms to handle cross-page relationships and ensure coherent content extraction.

**1. `Document` Class - Storage and Management:**

The `marker.schema.document.Document` class is central to managing the document's structure.
    - It holds a list of `PageGroup` objects, where each `PageGroup` represents a single page and contains its constituent blocks (text, images, tables, etc.). The `PageGroup` object (representing a page, defined in `marker.schema.groups.page.PageGroup`) stores an ordered list of these `BlockId` objects in its `structure` attribute. This list dictates the primary sequence of blocks on that page.
    - Each block within a page has a unique `BlockId`. This `BlockId` (defined in `marker.schema.blocks.base.BlockId`) is a Pydantic model containing `page_id` (integer, 0-indexed page number), `block_id` (an optional integer assigned sequentially as blocks are added to a page), and `block_type` (an optional `BlockTypes` enum). Its string representation is typically like `/page/0/Text/12`.
    - The `Document` class provides methods to retrieve specific pages (`get_page(page_id)`) and blocks (`get_block(block_id)`).

**2. Navigation Between Blocks and Pages:**

The `Document` class facilitates navigation across page boundaries:
    - `get_next_block(block, ignored_block_types)`: Finds the next logical block after the given `block`. If the current page has no more blocks, it seamlessly moves to the first relevant block on the subsequent page. It can ignore specified block types during this search.
    - `get_prev_block(block)`: Finds the previous block. If the current block is the first on the page, it moves to the last relevant block of the preceding page.
    - `get_next_page(page)`: Returns the `PageGroup` object for the page immediately following the current `page`.
    - `get_prev_page(page)`: Returns the `PageGroup` object for the page immediately preceding the current `page`.

These navigation methods are crucial for processors that need to understand context beyond a single page, such as identifying multi-page tables or continuous text sections.

**3. `OrderProcessor` - Block Reordering:**

The `marker.processors.order.OrderProcessor` plays a role in maintaining correct block order, especially when layout analysis might have "sliced" a page (i.e., processed it in multiple segments).
    - **Purpose:** If a page was sliced during layout detection (`page.layout_sliced` is True) and text was extracted directly from the PDF (`page.text_extraction_method == "pdftext"`), the `OrderProcessor` ensures blocks are sorted according to their original appearance in the PDF.
    - **Method:**
        - It calculates an "average vertical position" for each block based on the positions of its constituent text spans in the original PDF. The 'average vertical position' for a block containing text spans is calculated as `(spans[0].minimum_position + spans[-1].maximum_position) / 2`. The `minimum_position` and `maximum_position` attributes of a `Span` object (from `marker.schema.text.span.Span`) are integer values that represent the ordinal position of the span as extracted from the PDF's content stream, effectively indicating its reading order. Thus, this calculation provides a proxy for the block's vertical center based on its first and last text elements.
        - For blocks without direct text spans (e.g., images), their position is inferred based on their initial placement in the `page.structure` relative to text-containing blocks. The processor iterates through blocks needing an index. If a preceding block in the `page.structure` has a calculated index, the current block gets that index plus an increment. If no preceding block has an index, it looks for a succeeding block with an index and assigns its index minus an increment. This places non-text blocks logically between the text blocks they were initially positioned near.
        - Finally, it re-sorts the `page.structure` (the list of block IDs on the page) based on these calculated positions.
    - This processor helps correct potential ordering issues that might arise from segmented page processing, ensuring that the final output reflects the natural reading order.

**4. `SectionHeaderProcessor` - Cross-Page Header Identification:**

The `marker.processors.sectionheader.SectionHeaderProcessor` is responsible for identifying section headers and assigning appropriate heading levels (e.g., H1, H2). This process inherently handles headers that might appear at the beginning or end of pages.
    - **Identification:** It iterates through all blocks in the document and identifies those classified as `BlockTypes.SectionHeader`.
    - **Level Assignment:**
        - It collects the `line_heights` of all identified section headers across the entire document. The `line_heights` for each potential section header are calculated by the `block.line_height(document)` method (found in `marker.schema.blocks.base.Block`). This method returns the block's polygon height divided by the number of `BlockTypes.Line` objects it contains, effectively giving the average height of a single line of text within that header block.
        - It uses K-Means clustering (`sklearn.cluster.KMeans`) to group these line heights into a predefined number of levels (`level_count`, default: 4). It uses `sklearn.cluster.KMeans` with `n_clusters=num_levels` (default 4 levels), `random_state=0` (for reproducibility), and `n_init='auto'`. In recent scikit-learn versions, `n_init='auto'` means the K-Means algorithm runs multiple times with different initial centroid seeds to ensure a more stable and optimal clustering result, avoiding local minima.
        - Based on these clusters, it defines height ranges for each heading level. To define `heading_ranges`:
            1.  The mean line height for each of the `num_levels` clusters is calculated.
            2.  All detected header line heights are paired with their assigned cluster label and sorted by line height.
            3.  The code iterates through this sorted list. It attempts to group line heights into distinct ranges. A new range is started if the current line height belongs to a cluster whose mean is significantly different from the previous one (controlled by `merge_threshold`, though its application in the current loop appears complex and aims to separate clusters that are not close in terms of mean height).
            4.  The resulting list of `(min_height_in_range, max_height_in_range)` tuples is sorted in descending order (largest heights first).
            5.  Finally, each header block is assigned a level (1, 2, ...) by finding which range its `block_height` falls into (using `block_height >= min_height * self.height_tolerance`).
        - A `default_level` (default: 2) is assigned if a header's height doesn't clearly fit into the detected ranges or if the header is empty.
    - **Configuration:**
        - `level_count`: The number of distinct heading levels to identify.
        - `merge_threshold`: A parameter used in clustering to decide if distinct clusters of font sizes are close enough to be merged (not directly used in the final level assignment logic in the provided code, but part of the processor's attributes).
        - `height_tolerance`: A tolerance factor (default: 0.99) used when comparing a block's height to the minimum height of a heading range.
    - By analyzing headers globally, this processor ensures consistent heading level assignment even if, for example, an H2 heading is the first item on a new page.

These components and processors work together to provide a robust framework for handling documents where content and structure flow across multiple pages.
