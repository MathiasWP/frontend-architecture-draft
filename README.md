# Frontend architecture

This project contains a proposal for a cleaner frontend architecture/folder-setup. I've created example files in some of the folders to (hopefully) paint the picture a little better than just this readme.

Note that a lot of basic concepts are skipped here, e.g. that components should **only** care about UI related code and get as much data as possible as props. The goal here is to focus on the overall structure.

I have no strong opinions when it comes to conventions for the new layers. In the examples i've used camelCase for naming services and they are just functions that are exported together as a module. But there are many valid approaches here depending on your preferences.

## Key Concepts
- **Modular Architecture:** Emphasis on reusable and independent components.
- **Domain-Driven Design:** Organization of code aligns with business domains.
- **Layered Approach:** Separation of concerns across different layers - UI, Services, State Management.
- **CQRS (Command Query Responsibility Segregation):** Distinct handling of read (query) and write (command) operations.

## Folder Structure

From outermost to innermost layer

```bash
/src
  |-- /pages                 # Screen-level components.
  |-- /widgets               # Composite UI components aggregating features.
  |-- /features              # Standalone features, specific functionalities.
  |-- /components            # Shared, reusable UI components (prev. UI-kit)
  |-- /services              # Business logic, API interactions, socket management.
  |-- /store                 # Global state management based on view-models
  |-- /utils                 # Utility functions and helpers.
  |-- /models                # Data models and type definitions.
```

### Pages
Contains the top-level components representing entire screens or views in the application. Each page component orchestrates the composition of widgets and features.

### Widgets
Larger, composite components that combine various features and shared components. Widgets represent significant sections of the UI and can span across multiple business domains.

### Features
Self-contained, specific functionalities that are relatively independent. These can range from simple components like a User List to more complex ones like a File Uploader.

### Components
Reusable UI elements that are used across features, widgets, and pages. This includes items like buttons, input fields, and modals.

### Services
Handles all the business logic and data fetching operations. Organized by domains, each service is responsible for interactions with the backend and managing any related business rules. It includes:

- **Domain-specific services** (e.g., **criterionService**, **roleService**)
- **Generic services** (e.g., **socketService**, **authenticationService**)

### Store
Manages the global state of the application. Stores are split the same way we do with sendQuery, but it's handled on this layer instead.

### Utils
Contains general utility functions that are used throughout the application. This includes formatting functions, validators, and any other helper functions.

### Models
Defines the structure for various data entities in the application, aligning with the backend data models, cms data models and view-models for our UI. It ensures consistency in data handling and provides a clear understanding of data structures used across the frontend.

## Examples

These examples are simplified and focus mainly usage of the layers.

### Uploading a file that may have a name collision

```svelte
<script>
// features/documents/FileUpload.svelte
import fileService from '../services/file-service';
import { showConfirmDialog } from '../components/dialog';

async function handleFileUpload(file) {
    const exists = await fileService.checkFileExists(file.name);
    if (exists) {
        const userConsent = await showConfirmDialog('File exists. Do you want to copy this file?');
        if (userConsent) {
        await fileService.uploadFile(file, { copy: true });
        }
    } else {
        await fileService.uploadFile(file);
    }
}
</script>
```

```ts
// services/file-service.ts
import { get } from 'svelte/store';
import { getDocumentStore } from '../stores/document-store';

function checkFileExists(projectId, fileName) {
    // Stores are split up by a specified identifier instead of hashing the payload,
    // so we can retriece the specific store for the documents on this project
    const documentStore = getDocumentStore(projectId)
    return get(documentStore).some(existingDocument => existingDocument.name === fileName)
}
```

### Showing errors to users

*Using an alertService to display a toast in the UI is a viable and often preferred approach in modern web applications. It does not necessarily break the layer rules if implemented thoughtfully. The key is to maintain a clear separation of concerns and ensure that the service layer doesn't directly manipulate the UI, but rather communicates state or intentions that the UI layer can then interpret and display.*

```ts
// services/alert-service.js

let onShowError;

export const alertService = {
    registerShowErrorCallback(callback) {
        onShowError = callback;
    },

    showError(message) {
        if (onShowError) {
            onShowError(message);
        }
    },

    // ... other methods for different types of alerts
};
```

```svelte
<script>
// Toast.svelte
import { onMount } from 'svelte';
import alertService from '../services/alertService';
import Toast from '../components/Toast.svelte';

let toastMessage = '';
let showToast = false;

function showErrorToast(message) {
    toastMessage = message;
    showToast = true;
}

onMount(() => {
    alertService.registerShowErrorCallback(showErrorToast);
});
</script>

{#if showToast}
  <Toast message={toastMessage} />
{/if}
```

```svelte
<script>
// SomeComponent.svelte
import alertService from '../services/alert-service';
import loggingService from '../services/logging-service';

function handleSubmit() {
    try {
        // form submission logic
    } catch (error) {
        alertService.showError('Failed to submit form');
        loggingService.logError(error)
    }
}
</script>
```

### Fetching data and creating view models

```ts
// services/task-service.js
import type { TaskViewModel } from '../models/view-models/task';
import { derived } from 'svelte/store';
import socketService from './socket-service';
import cmsService from './cms-service';

async function fetchTasks(projectId) {
    const emptyState = { /* ... */ }
    // Reponse is equal to what we get from our existing sendQuery method.
    // The store is automatically updated on QueryResultChanged
    const queryStore = socketService.sendQuery('/tasks/get-task', {
        projectId
    }, emptyState)
    return queryStore
}

async function createTaskViewModelStore(projectId) {
    const taskStore = await fetchTasks(projectId);
    const taskContent = await cmsService.getTaskContent(projectId);

    return derived<TaskViewModel[]>(taskStore, $taskStore => {
        return {
            ...$taskStore,
            title: taskContent.title,
            description: taskContent.description,
        };
    });
}

export * as taskService
```

### Optimistic updates

Note: This is more a proof of concept than a working example.

```ts
// services/document-service.js
import type TaskViewModel from '../models/view-models/task';
import { derived } from 'svelte/store';
import { getDocumentStore } from '../stores/document-store';

async function uploadDocument(projectId, documentName, documentId) {
    const store = getDocumentStore(projectId);

    // This optimistic update does not cover the edge cases of receiving
    // multiple QueryResultChanged events that overwrite optimistic state with old data.
    store.update(documents => {
        documents.push({documentName, documentId})
        return documents
    })

    const response = await socketService.sendCommand('/documents/upload', {
        documentName,
        documentId
    })
    
    if(!response.ok) {
        // Undo update
    }
}

export * as documentService
```

### Feature flagging

```svelte
<script>
import { featureFlagService } from '../services/feature-flag-service';

const isSomeFeatureEnabled = featureFlagService.isEnabled('someFeature');
</script>

{#if $isSomeFeatureEnabled}
    <SomeFeatureComponent />
{/if}
```

### Tracking

```svelte
<script>
import trackingService  from '../services/tracking-service';

trackingService.setContext('some-context');

const track = trackingService.createTracker();

function handleButtonClick() {
    track('button_click');
}
</script>

<button on:click={handleButtonClick}>Click Me</button>
```