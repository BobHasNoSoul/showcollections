# showcollections
this is for jellyfin and is a mod that gives you the ability to see what other movies are in the same collection (same for tv shows etc)


this works on 10.10.7 

does this work for everything? not exactly currently the bug exists that it will not repopulate when jumping from item to item pages.. however if refreshed it works, if you go back to the home page it should also function as intended.. if you put any page between the item pages it appears to work normally i am aware of this bug and should fix this in the future but wanted to not leave a redditor hanging with this so pushed the beta out.. this is the beta

you will need to simply edit 

`itemDetails-index-html.ca5f15ff794311af00a6.chunk.js`

under the only line in there create a new line and add the following 

```
(function() {
    'use strict';

    let currentItemId = null;
    let userId = null;
    let token = null;
    let domain = window.location.origin;
    let isProcessing = false;
    let observer = null;
    let lastDescriptionText = null;
    let descriptionSelector = "div.page:nth-child(2) > div:nth-child(3) > div:nth-child(2) > div:nth-child(1) > div:nth-child(1) > div:nth-child(1) > div:nth-child(3) > p:nth-child(3) > bdi:nth-child(1) > p:nth-child(1)";

    // Get credentials from local storage
    const getCredentials = () => {
        console.debug("Fetching credentials from local storage");
        const creds = localStorage.getItem("jellyfin_credentials");
        if (!creds) {
            console.error("Could not parse jellyfin credentials");
            return null;
        }
        try {
            const parsed = JSON.parse(creds);
            const server = parsed.Servers[0];
            if (!server) {
                console.error("Could not find credentials for the client");
                return null;
            }
            console.debug("Credentials fetched: userId=", server.UserId);
            return { token: server.AccessToken, userId: server.UserId };
        } catch (e) {
            console.error("Could not parse jellyfin credentials", e);
            return null;
        }
    };

    // Parse item ID from URL hash
    const getItemIdFromUrl = () => {
        const hash = window.location.hash;
        const match = hash.match(/#\/details\?id=([a-f0-9]+)/);
        const itemId = match ? match[1] : null;
        console.debug(`Parsed URL: ${hash}, Item ID: ${itemId}`);
        return itemId;
    };

    // Fetch API helper
    const apiFetch = async (endpoint) => {
        console.debug(`Fetching API: ${domain}${endpoint}`);
        try {
            const response = await fetch(`${domain}${endpoint}`, {
                headers: { "X-Emby-Token": token }
            });
            if (!response.ok) throw new Error(`API error: ${response.status}`);
            const data = await response.json();
            console.debug(`API response received for ${endpoint}`);
            return data;
        } catch (e) {
            console.error("API fetch error:", e);
            return null;
        }
    };

    // Create or update the collection section
    const createCollectionSection = () => {
        console.debug("Creating collection section");
        let existingSection = document.getElementById("collectionItemsContainer");
        if (existingSection) {
            console.debug("Removing existing collection section");
            existingSection.remove();
        }

        let collectionContainer = document.createElement("div");
        collectionContainer.id = "collectionItemsContainer";
        collectionContainer.style.marginTop = "20px";

        let header = document.createElement("h2");
        header.innerText = "Other items in this collection";
        collectionContainer.appendChild(header);

        let itemsContainer = document.createElement("div");
        itemsContainer.id = "collectionItems";
        itemsContainer.style.display = "flex";
        itemsContainer.style.flexWrap = "wrap";
        itemsContainer.style.gap = "10px";
        collectionContainer.appendChild(itemsContainer);

        const selectors = [
            ".detailSectionContent",
            ".mainAnimatedPages",
            ".pageContainer",
            "body"
        ];

        let targetElement = null;
        for (const selector of selectors) {
            targetElement = document.querySelector(selector);
            if (targetElement) {
                console.debug(`Found target element: ${selector}`);
                break;
            }
        }

        if (targetElement) {
            const parent = targetElement.tagName === "BODY" ? targetElement : targetElement.parentNode;
            parent.appendChild(collectionContainer);
            console.debug("Collection section inserted into DOM");
        } else {
            console.warn("No target element found for insertion");
        }

        return itemsContainer;
    };

    // Fetch and display collection items
    const fetchCollectionItems = async (itemId) => {
        if (isProcessing) {
            console.debug("Processing in progress, skipping");
            return;
        }
        isProcessing = true;
        console.debug(`Fetching collection items for item ID: ${itemId}`);
        currentItemId = itemId;

        let retries = 3;
        let itemsContainer = null;
        while (retries > 0 && !itemsContainer) {
            itemsContainer = createCollectionSection();
            if (!itemsContainer) {
                console.debug(`Retrying DOM insertion, attempts left: ${retries}`);
                await new Promise(resolve => setTimeout(resolve, 500));
                retries--;
            }
        }

        if (!itemsContainer) {
            console.error("Failed to create collection section after retries");
            isProcessing = false;
            return;
        }

        itemsContainer.innerHTML = "";

        const boxsetsResponse = await apiFetch(`/Items?UserId=${userId}&IncludeItemTypes=BoxSet&Recursive=true&SortBy=SortName&SortOrder=Ascending`);
        if (!boxsetsResponse || !boxsetsResponse.Items) {
            console.debug("No boxsets found");
            itemsContainer.innerText = "No collections found.";
            isProcessing = false;
            return;
        }

        const boxsets = boxsetsResponse.Items;
        console.debug(`Found ${boxsets.length} boxsets`);

        let foundItems = false;
        for (const boxset of boxsets) {
            const childrenResponse = await apiFetch(`/Items?UserId=${userId}&ParentId=${boxset.Id}`);
            if (!childrenResponse || !childrenResponse.Items) {
                console.debug(`No children found for boxset ${boxset.Id}`);
                continue;
            }

            const children = childrenResponse.Items;
            const isInCollection = children.some(child => child.Id === itemId);
            if (isInCollection) {
                console.debug(`Item ${itemId} found in boxset ${boxset.Name}`);
                const otherItems = children.filter(child => child.Id !== itemId);
                if (otherItems.length > 0) {
                    foundItems = true;
                    const header = document.querySelector("#collectionItemsContainer h2");
                    header.innerText = `Other items in "${boxset.Name}"`;

                    otherItems.forEach(item => {
                        const card = document.createElement("div");
                        card.style.width = "150px";
                        card.style.textAlign = "center";

                        const link = document.createElement("a");
                        link.href = `#/details?id=${item.Id}&serverId=${new URLSearchParams(window.location.hash.slice(1)).get('serverId')}`;
                        link.style.textDecoration = "none";
                        link.style.color = "inherit";

                        const img = document.createElement("img");
                        img.src = `${domain}/Items/${item.Id}/Images/Primary?tag=${item.ImageTags.Primary || ''}&quality=90`;
                        img.style.width = "100%";
                        img.style.height = "auto";
                        img.style.borderRadius = "5px";
                        img.style.objectFit = "contain";
                        img.alt = item.Name;
                        link.appendChild(img);

                        const name = document.createElement("div");
                        name.textContent = item.Name;
                        name.style.marginTop = "5px";
                        link.appendChild(name);

                        card.appendChild(link);
                        itemsContainer.appendChild(card);
                    });
                    console.debug(`Added ${otherItems.length} items to collection section`);
                    break;
                }
            }
        }

        if (!foundItems) {
            console.debug("Item not found in any collection");
            itemsContainer.innerText = "This item is not in any collection.";
        }
        isProcessing = false;
    };

    // Unload collection section
    const unloadCollectionSection = () => {
        if (isProcessing) {
            console.debug("Processing in progress, skipping unload");
            return;
        }
        console.debug("Unloading collection section");
        const collectionContainer = document.getElementById("collectionItemsContainer");
        if (collectionContainer) {
            collectionContainer.remove();
        }
        currentItemId = null;
    };

    // Debounce function
    const debounce = (func, wait) => {
        let timeout;
        return (...args) => {
            clearTimeout(timeout);
            timeout = setTimeout(() => func.apply(this, args), wait);
        };
    };

    // Handle URL changes
    const handleUrlChange = () => {
        const newItemId = getItemIdFromUrl();
        console.debug(`Handling URL change, new item ID: ${newItemId}, current item ID: ${currentItemId}`);
        if (newItemId && newItemId !== currentItemId) {
            fetchCollectionItems(newItemId);
            setupMutationObserver(); // Reattach observer after navigation
        } else if (!newItemId && currentItemId) {
            unloadCollectionSection();
        }
    };

    // Check description text and trigger reload if changed
    const checkDescriptionText = () => {
        const descriptionElement = document.querySelector(descriptionSelector);
        const currentText = descriptionElement ? descriptionElement.textContent.trim() : "";
        console.debug(`Checking description text: current="${currentText}", last="${lastDescriptionText}"`);
        if (descriptionElement && currentText !== lastDescriptionText) {
            console.debug("Description text changed, reinitializing");
            lastDescriptionText = currentText;
            handleUrlChange();
        }
        return !!descriptionElement; // Return true if element exists
    };

    // Set up MutationObserver for the description element
    const setupMutationObserver = () => {
        // Disconnect any existing observer
        if (observer) {
            observer.disconnect();
            console.debug("Disconnected existing MutationObserver");
        }

        // Target a stable parent to handle SPA element replacements
        const targetNode = document.querySelector(".detailSectionContent") || document.querySelector(".detailPageContent") || document.body;
        if (!targetNode) {
            console.warn("No stable parent found (.detailSectionContent, .detailPageContent, or body), retrying in 500ms");
            setTimeout(setupMutationObserver, 500);
            return;
        }

        // Initialize lastDescriptionText
        const descriptionElement = document.querySelector(descriptionSelector);
        lastDescriptionText = descriptionElement ? descriptionElement.textContent.trim() : "";
        console.debug(`Initial description text: ${lastDescriptionText}`);

        observer = new MutationObserver(debounce(() => {
            console.debug("Mutation detected in parent, checking description text");
            checkDescriptionText();
        }, 100));

        observer.observe(targetNode, {
            childList: true, // Observe additions/removals of child nodes
            characterData: true, // Observe text content changes
            subtree: true // Observe changes in descendants
        });
        console.debug(`MutationObserver set up on ${targetNode.tagName}${targetNode.className ? '.' + targetNode.className : ''}`);
    };

    // Override fetch to intercept /Items/{itemId} requests
    const originalFetch = window.fetch;
    window.fetch = async function(input, init) {
        const response = await originalFetch.call(this, input, init);
        let url = input instanceof Request ? input.url : input;

        // Normalize URL for relative paths
        if (!url.startsWith("http")) {
            url = new URL(url, domain).href;
        }

        // Match /Items/{itemId} requests
        const match = url.match(/\/Items\/([a-f0-9]+)(\?|$)/);
        if (match) {
            const newItemId = match[1];
            console.debug(`Detected API request for item ID: ${newItemId}`);
            if (newItemId && newItemId !== currentItemId) {
                console.debug(`New item ID detected in API request, triggering fetchCollectionItems`);
                setTimeout(() => {
                    fetchCollectionItems(newItemId);
                    setupMutationObserver(); // Reattach observer
                }, 100); // Debounce to avoid rapid triggers
            }
        }

        return response;
    };

    // Fallback polling mechanism
    const startPolling = () => {
        setInterval(() => {
            if (!checkDescriptionText()) {
                console.debug("Description element not found, reattaching observer");
                setupMutationObserver();
            }
            const newItemId = getItemIdFromUrl();
            if (newItemId && newItemId !== currentItemId) {
                console.debug(`Polling detected new item ID: ${newItemId}`);
                fetchCollectionItems(newItemId);
                setupMutationObserver();
            }
        }, 1000); // Check every 1 second
    };

    // Initialize
    const init = () => {
        console.debug("Initializing script");
        const creds = getCredentials();
        if (!creds) {
            console.error("Initialization failed: No credentials");
            return;
        }
        token = creds.token;
        userId = creds.userId;

        handleUrlChange();
        window.addEventListener('hashchange', debounce(handleUrlChange, 300));
        setupMutationObserver();
        startPolling(); // Start fallback polling
    };

    // Delay initialization by 0.5 seconds
    const delayedInit = () => {
        console.debug("Scheduling initialization with 0.5s delay");
        setTimeout(() => {
            console.debug("Executing delayed initialization");
            init();
        }, 500);
    };

    // Run delayed init when DOM is ready
    if (document.readyState === 'complete' || document.readyState === 'interactive') {
        console.debug("DOM ready, scheduling delayed init");
        delayedInit();
    } else {
        document.addEventListener('DOMContentLoaded', () => {
            console.debug("DOMContentLoaded, scheduling delayed init");
            delayedInit();
        });
    }
})();
```
save it and clear the cache and reload you will then see the glorious new addition on the items pages 

<img width="1920" height="2930" alt="Screenshot 2025-09-04 at 20-35-15 BlueBoxofDOOM" src="https://github.com/user-attachments/assets/234101b5-917c-4c15-8f23-8f9c2a8eeb34" />

