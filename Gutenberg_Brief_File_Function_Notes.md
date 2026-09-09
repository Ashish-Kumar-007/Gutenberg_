# Gutenberg Architecture — Brief File + Function Notes

> Source basis: the supplied architecture HTML files. This note keeps every file-card and function-card entry, but compresses each description into a brief study format without replacing source terminology.


---

# packages/data — full file & function map

**Owns:**
- The registry abstraction — an isolated container holding many named stores (registries can nest for scoped editor contexts).
- The store descriptor ( createReduxStore ) and the live Redux store instances it produces.
- The metadata / resolver subsystem that tracks selector resolution state and drives async data fetching.
- The middleware stack (promise, thunk, resolvers-cache) and the built-in controls .
- The React bindings : useSelect , useDispatch , useRegistry , providers, and HOCs.
- The persistence plugin for saving store state to browser storage.

## File map / file details

- **src/index.ts ★ ENTRY** — Purpose: The package entry point. Re-exports all public APIs (hooks, HOCs, providers, createRegistry , createReduxStore , createSelector , controls , plugins ) and defines the global convenience functions ( select , dispatch , resolveSelect , suspendSelect , subscribe , register , use , and deprecated registerStore / registerGenericStore ) that all delegate to the default registry.
- **src/registry.ts ★ CORE** — Purpose: Contains createRegistry() , the heart of the data module. Produces a DataRegistry : an isolated orchestrator that manages store registration, selection, dispatch, subscription, batching, private-API inheritance between parent/child registries, and plugin extension.
- **src/factory.ts** — Purpose: Factories for registry-aware primitives: createRegistrySelector (a selector that can reach other stores) and createRegistryControl (a control that receives the registry).
- **src/select.ts · src/dispatch.ts** — Purpose: The global select(store) and dispatch(store) convenience functions. They simply delegate to the default registry — the warning is they only work with the default registry, not a custom RegistryProvider .
- **src/create-selector.ts** — Purpose: Provides createSelector , a memoization utility for selectors — a re-export of rememo so consumers don't need to depend on rememo directly.
- **src/default-registry.ts** — Purpose: Creates and exports the single default DataRegistry via createRegistry() . Every global convenience function in index.ts delegates to it.
- **src/controls.ts** — Purpose: Defines the built-in control descriptors ( select , resolveSelect , dispatch ) and the handler implementations used inside generator-based action creators (resolvers).
- **src/lock-unlock.ts** — Purpose: Opts into the private-APIs system and exports the lock / unlock functions used to mark private data (private actions/selectors, store registration functions) inaccessible from non-core modules.
- **src/promise-middleware.ts · src/resolvers-cache-middleware.ts** — Purpose: Two Redux middlewares. promise-middleware resolves Promise-valued actions; resolvers-cache-middleware invalidates resolver caches when dispatched actions match a resolver's shouldInvalidate rule.
- **src/types.ts** — Purpose: All TypeScript interfaces and type aliases: StoreDescriptor , ReduxStoreConfig , DataRegistry , StoreInstance , MapSelect , ThunkArgs , conditional types for curried selectors/actions, and resolution state types.
- **src/redux-store/index.ts ★ CORE** — Purpose: createReduxStore(key, options) creates a store descriptor that, when instantiated into a registry, produces a real Redux store enhanced with the metadata reducer, the middleware stack, resolver integration, bound actions, and bound selectors. This file contains the most intricate binding logic in the package.
- **src/redux-store/combine-reducers.ts** — Purpose: A custom combineReducers that (unlike Redux's) does not validate reducers and returns the old state object when nothing changed (referential stability).
- **src/redux-store/keyed-reducer.ts** — Purpose: keyedReducer(actionProperty) — a higher-order reducer that partitions state by a property on the action. Used by the metadata reducer to key resolution state by selector name.
- **src/redux-store/thunk-middleware.ts** — Purpose: A Redux middleware that intercepts function (thunk) actions and calls them with the thunk argument object ( { registry, dispatch, select, resolveSelect } ).
- **src/redux-store/metadata/ ★ subsystem** — Purpose: The resolver / resolution-tracking subsystem. Tracks the status of every selector+args resolution ( resolving / finished / error ) via a separate metadata reducer slice, and exposes metadata selectors/actions injected into every store. Files: actions.ts , reducer.ts , selectors.ts , types.ts , utils.ts .
- **src/components/use-select/ ★ hook** — Purpose: useSelect and useSuspenseSelect . The primary hooks for subscribing to store state, supporting both a "static select" mode (pass a store descriptor) and a "mapping select" mode (pass a callback). Uses useSyncExternalStore and a shared render priority queue.
- **src/components/use-dispatch/ · with-dispatch/ · with-select/ · with-registry/** — Purpose: useDispatch returns bound action creators; its HOC counterparts withDispatch , withSelect , and withRegistry inject store-derived props into wrapped components.
- **src/components/registry-provider/ · async-mode-provider/** — Purpose: Context providers. RegistryProvider / RegistryConsumer provide a specific DataRegistry ( useRegistry reads it); AsyncModeProvider switches useSelect into deferred (idle-priority) re-rendering.
- **src/plugins/persistence/ ★ plugin** — Purpose: A data plugin that persists store state to browser storage (localStorage by default), enabled per-store via the persist option. Includes the storage backends in storage/ ( default.ts , object.ts ).
- **src/store/index.ts** — Purpose: Defines and registers the package's own core/data meta-store (which holds the list of registered/available stores), automatically bootstrapped into every new registry.
- **src/utils/emitter.ts** — Purpose: createEmitter() — a minimal event emitter with pause/resume support used for batching notifications (the global registry emitter and each store's per-store emitter).

## Function notes

### `createRegistry( storeConfigs?, parent? )` — · src/registry.ts
- Purpose / why used: Creates a complete, independent DataRegistry . This is the top of the data architecture — everything else hangs off a registry.
- When used: Once per editor context. The default registry is created here (via default-registry.ts ), and child registries (e.g. the site editor's pattern scope) call it with a parent .
- Who uses it: default-registry.ts and any code that scopes state with a child registry. Its returned object is the DataRegistry consumed by all React bindings.
- Inputs: optional storeConfigs (record of pre-registered Redux store configs) and an optional parent registry. Output: the DataRegistry object (with batch , subscribe , select , resolveSelect , suspendSelect , dispatch , use , register , and the deprecated registerStore / registerGenericStore ).

### `registry.register( storeDescriptor )` — · registry object method (registry.ts)
- Purpose / why used: Registers a store by calling storeDescriptor.instantiate(registry) and storing the resulting InternalStoreInstance . Handles propagation of private actions/selectors from the parent registry.
- When used: Whenever a store is registered against a registry (the global convenience register() in index.ts delegates here).
- Who uses it: The public register API, registerStore , other packages' store modules (e.g. core/blocks, core/editor).
- Inputs: a StoreDescriptor ( { name, instantiate } ). Output: none — adds to stores and wires it to the global emitter.

### `registry.select( storeNameOrDescriptor )` — · registry object method (registry.ts)
- Purpose / why used: Returns an object of bound selectors for the named store (state pre-bound). Falls back to the parent registry if not found locally.
- When used: Anywhere code reads store state — inside components, resolvers, or the global select() .
- Who uses it: The global select() , useSelect (via __unstableMarkListeningStores ), resolvers.
- Inputs: store name string or store descriptor. Output: CurriedSelectorsOf<S> — a map of selector functions.

### `registry.dispatch( storeNameOrDescriptor )` — · registry object method (registry.ts)
- Purpose / why used: Returns an object of bound action creators for the named store. Falls back to the parent registry.
- When used: Anywhere code dispatches an action.
- Who uses it: The global dispatch() , useDispatch , withDispatch .
- Inputs: store name string or descriptor. Output: ActionCreatorsOf<S> .

### `registry.resolveSelect / registry.suspendSelect` — · registry object methods (registry.ts)
- Purpose / why used: Like select , but return promise-wrapped selectors ( resolveSelect ) or suspense-wrapping selectors ( suspendSelect ). They wait for the resolver to complete before resolving.
- When used: resolveSelect in resolvers and async logic; suspendSelect in React Suspense flows.
- Who uses it: The global resolveSelect() / suspendSelect() , useSuspenseSelect , the @@data/RESOLVE_SELECT control.
- Inputs: store name. Output: promise/suspense-wrapped selectors via store.getResolveSelectors() / store.getSuspendSelectors() .

### `registry.batch( callback ) / registry.subscribe( listener, storeName? )` — · registry object methods (registry.ts)
- Purpose / why used: batch pauses all emitters during a callback and resumes after, coalescing multiple state changes into one notification cycle. subscribe listens to all stores (global emitter) or a specific store.
- When used: batch around multi-dispatch operations; subscribe for external change listeners.
- Who uses it: useSelect 's subscriber, external integrations.
- Inputs: a callback; a listener (+ optional store). Output: subscribe returns an unsubscribe function.

### `registry.use( plugin, options? )` — · registry object method (registry.ts)
- Purpose / why used: ( Deprecated ) Extends the registry by merging in the return value of a plugin function.
- When used: Legacy plugin authors extending the registry; the persistence plugin exposes a registerStore override this way.
- Who uses it: The persistence plugin mechanism.
- Inputs: a DataPlugin function (+ options). Output: the extended registry.

### `createRegistrySelector( registrySelector )` — · src/factory.ts
- Purpose / why used: Creates a selector that can select from other stores . The outer function receives the registry's select method and returns a normal state selector. Results are cached per registry in a WeakMap .
- When used: When authoring a selector for store A that reads store B's state.
- Who uses it: Store authors (e.g. core/editor selectors that read core/block-editor ). The isRegistrySelector flag lets the binding system wrap it.
- Inputs: (select) => (state, ...args) => result . Output: the wrapped selector.

### `createRegistryControl( registryControl )` — · src/factory.ts
- Purpose / why used: Creates a control function that receives the registry as its first curried argument. The isRegistryControl = true flag tells instantiateReduxStore to call control(registry) to obtain the real control handler.
- When used: When authoring controls that need registry access.
- Who uses it: The built-in controls in controls.ts , and package-specific controls.
- Inputs: (registry) => (...args) => result . Output: the control with the flag set.

### `controls (select / resolveSelect / dispatch) + builtinControls` — · src/controls.ts
- Purpose / why used: The built-in control descriptors (used with yield inside generator action creators) and their handlers . Handlers are built with createRegistryControl .
- When used: Inside resolvers/actions: yield controls.select(...) , yield controls.resolveSelect(...) , yield controls.dispatch(...) .
- Who uses it: All generator-based actions and resolvers across the editor.
- Inputs: store + selectorName/actionName + args. Output: a ControlDescriptor object { type, storeKey, selectorName?, actionName?, args } .

### `createReduxStore( key, options )` — · src/redux-store/index.ts
- Purpose / why used: Creates a store descriptor ( { name, instantiate } ) — not yet wired to a registry. On instantiate(registry) it builds the full live Redux store with the metadata reducer, the middleware stack, resolver integration, and bound actions/selectors.
- When used: By every store definition in the editor (core, blocks, editor, block-editor, edit-post, etc.).
- Who uses it: All package store modules; the global registerStore convenience.
- Inputs: key (store name), options ( ReduxStoreConfig : reducer , actions , selectors , resolvers , controls , initialState ). Output: a StoreDescriptor .

### `instantiateReduxStore( key, options, registry, thunkArgs )` — · src/redux-store/index.ts (private)
- Purpose / why used: Creates the actual Redux store. Builds the middleware chain ( resolversCache → promise → reduxRoutine(controls) → thunk ) and the combined { metadata, root } reducer, then returns the store.
- When used: Inside createReduxStore 's instantiate .
- Who uses it: createReduxStore .
- Inputs: key, options, registry, thunkArgs. Output: the Redux store (with __unstableOriginalGetState preserved).

### `bindSelector / bindMetadataSelector / mapResolveSelector / mapSuspendSelector` — · src/redux-store/index.ts (internal)
- Purpose / why used: The binding system that turns raw selector functions into the bound selectors returned by registry.select() . Selectors with resolvers are wrapped so calling them also triggers the resolver; metadata selectors read state.metadata ; resolve/suspend variants wrap results in promises or Suspense.
- When used: Inside instantiate , once per selector.
- Who uses it: The store instantiation pipeline.
- Inputs: a selector (+ name), or a metadata selector. Output: a bound selector function (with hasResolver etc. flags).

### `metadata actions (start/finish/fail/invalidate resolution...)` — · src/redux-store/metadata/actions.ts
- Purpose / why used: Action creators injected into every store to track resolver/resolution state. Covers single and batch variants, plus invalidation (per selector+args, per selector name, or whole store).
- When used: Automatically dispatched by the resolver binding pipeline; invalidate actions are dispatched by the resolvers-cache middleware or manually.
- Who uses it: mapSelectorWithResolver , createResolversCacheMiddleware , wholesale store invalidation callers.
- Inputs: selectorName + args (+ error for fail). Output: action objects like { type: 'START_RESOLUTION', selectorName, args } .

### `metadata reducer (isResolved / subKeysIsResolved)` — · src/redux-store/metadata/reducer.ts
- Purpose / why used: Reducer tracking the resolution status of each selector+args combo. Uses keyedReducer('selectorName') to partition by selector name, and EquivalentKeyMap for argument keys (so equivalent args share a key).
- When used: On every resolution action.
- Who uses it: Redux (part of the metadata slice). Read by the metadata selectors.
- Inputs: state, action. Output: new resolution state.

### `metadata selectors (getResolutionState / hasStartedResolution / isResolving ...)` — · src/redux-store/metadata/selectors.ts
- Purpose / why used: Query the resolution state: hasStartedResolution , hasFinishedResolution , hasResolutionFailed , getResolutionError , isResolving , getCachedResolvers , hasResolvingSelectors , countSelectorsByStatus , and the deprecated getIsResolving . Returned on every bound select(store) object.
- When used: Anywhere code needs to know whether/why data is loading.
- Who uses it: The resolver binding code, UI loading indicators, and the resolvers-cache middleware ( getCachedResolvers ).
- Inputs: state, selectorName, args. Output: booleans / state / error (varies).

### `combineReducers( reducers )` — · src/redux-store/combine-reducers.ts
- Purpose / why used: Custom reducer combiner with referential stability (returns old state if no slice changed).
- When used: Wherever combineReducers is imported from @wordpress/data .
- Who uses it: Store definitions and the metadata/root combiner.
- Inputs: a map of reducers. Output: a combined reducer.

### `keyedReducer( actionProperty )` — · src/redux-store/keyed-reducer.ts
- Purpose / why used: Higher-order reducer that partitions state by a property on the action. If action[actionProperty] is undefined, state is returned unchanged.
- When used: By the metadata reducer to key resolution state by selector name.
- Who uses it: metadata/reducer.ts .
- Inputs: an action property name. Output: a wrapped reducer.

### `createThunkMiddleware( args ) / promiseMiddleware` — · src/redux-store/thunk-middleware.ts · src/promise-middleware.ts
- Purpose / why used: createThunkMiddleware intercepts function (thunk) actions and invokes them with the thunk args object. promiseMiddleware resolves actions that are Promises.
- When used: In the store middleware stack.
- Who uses it: The store pipeline; thunks dispatch other actions via the passed-in methods.
- Inputs: thunk args / a Redux store. Output: middleware.

### `useSelect( mapSelect, deps? )` — · src/components/use-select/index.ts
- Purpose / why used: The primary hook for subscribing to store state. Supports two modes: pass a store descriptor to get all selectors (static mode, no subscription), or a mapping function (select, registry) => data for derived data (re-renders when derived data changes).
- When used: In every component that reads editor/store state.
- Who uses it: All editor components; withSelect wraps it internally.
- Inputs: mapSelect (callback or store descriptor), optional deps . Output: the derived value (or bound selectors).

### `_useMappingSelect / Store factory / useSuspenseSelect` — · src/components/use-select/index.ts (internal)
- Purpose / why used: The machinery behind useSelect . The Store factory returns { subscribe, getValue } consumed by useSyncExternalStore . It runs mapSelect under __unstableMarkListeningStores to record which stores it reads, subscribes to those stores, and keeps the value shallow-stable. In async mode, listener calls go through a shared render priority queue.
- When used: Every useSelect call; useSuspenseSelect is the suspense=true variant.
- Who uses it: useSelect , useSuspenseSelect .
- Inputs: mapSelect, isAsync (and suspense flag). Output: subscribe/getValue pair.

### `useDispatch( storeNameOrDescriptor? ) / useDispatchWithMap( dispatchMap, deps )` — · src/components/use-dispatch/use-dispatch.ts · use-dispatch-with-map.ts
- Purpose / why used: useDispatch returns bound action creators for a store (or the raw registry.dispatch if no store is given). useDispatchWithMap builds stable dispatch props from a mapping function (used by withDispatch ).
- When used: In any component that needs to dispatch actions.
- Who uses it: All editor components; withDispatch .
- Inputs: store name/descriptor, or a map function. Output: bound action creators / props object.

### `withSelect / withDispatch / withRegistry` — · components/with-select · with-dispatch · with-registry
- Purpose / why used: Higher-order components that inject store-derived props. withSelect wraps a mapSelectToProps callback into useSelect ; withDispatch uses useDispatchWithMap ; withRegistry injects the registry via RegistryConsumer .
- When used: In class components (or legacy code) that need store access.
- Who uses it: Many editor HOCs and plugins.
- Inputs: a map function. Output: a HigherOrderComponent.

### `RegistryProvider / RegistryConsumer / useRegistry / AsyncModeProvider / useAsyncMode` — · components/registry-provider · async-mode-provider
- Purpose / why used: Context plumbing. RegistryProvider / RegistryConsumer and useRegistry read the current DataRegistry ; AsyncModeProvider and useAsyncMode switch useSelect into deferred re-rendering.
- When used: At the root of scoped editor contexts and to opt into async rendering.
- Who uses it: useSelect , useDispatch , withRegistry , the site editor's pattern scope.
- Inputs: context value. Output: the registry or a boolean.

### `createSelector( selector, getDependants? )` — · src/create-selector.ts
- Purpose / why used: Memoizes a selector's return value. The cache is kept while all entries in the dependants array stay referentially equal. Re-exports rememo .
- When used: When authoring selectors that compute derived data.
- Who uses it: Store selector authors across the editor.
- Inputs: a selector and an optional dependants-getter. Output: the memoized selector (with getDependants and clear ).

### `persistencePlugin / withLazySameState / createPersistenceInterface` — · src/plugins/persistence/index.ts
- Purpose / why used: Persists store state to browser storage (localStorage by default). It overrides registry.registerStore so that stores opt in via a persist option — either true (whole state) or an array of keys to track.
- When used: Registered as a data plugin for stores that want state survival across reloads.
- Who uses it: Legacy stores that opt in; the plugins.persistence export.
- Inputs: registry + plugin options ( storage , storageKey ). Output: a partial-registry override of registerStore .

### `createEmitter() / selectorArgsToStateKey( args )` — · src/utils/emitter.ts · src/redux-store/metadata/utils.ts
- Purpose / why used: createEmitter is the pause/resume event emitter behind batching. selectorArgsToStateKey normalizes selector argument arrays (null/undefined → [] , trailing undefined stripped) so f(1) and f(1, undefined) share a cache key.
- When used: In the registry and store emitters; in the metadata subsystem.
- Who uses it: createRegistry , store instantiation, metadata actions.
- Inputs: (emitter) nothing; (args) an arguments array. Output: an emitter / a key array.


---

# packages/blocks — full file & function map

**Owns:**
- The block type registry ( core/blocks store) and the public registerBlockType API.
- The block object factory ( createBlock , cloneBlock , ...) and the transform engine ( switchToBlockType ).
- The parser pipeline ( parse ) — HTML → block tree.
- The serializer ( serialize / serializeBlock ) — block tree → HTML with comment delimiters.
- Block validation ( validateBlock , isValidBlockContent ) and deprecation handling.
- Raw/paste handling transforms arbitrary HTML into blocks ( rawHandler , pasteHandler ).
- Templates , categories , and the children / node BlockNode helpers.

## File map / file details

- **src/index.ts ★ ENTRY** — Purpose: The package entry point. Re-exports the whole public API surface from ./api , the core/blocks store, deprecated.ts , and the TypeScript types. Also declares and locks the privateApis object ( isContentBlock , fieldsKey , formKey , editableRootKey , parseRawBlock ).
- **src/deprecated.ts · src/lock-unlock.ts · src/types.ts** — Purpose: deprecated.ts holds backward-compat re-exports; lock-unlock.ts opts into the private-APIs system; types.ts defines the core type shapes ( Block , BlockType , BlockTransform , BlockVariation , etc.).
- **src/api/index.ts ★ barrel** — Purpose: The barrel that assembles the entire public API: the factory, parser, raw handling, serializer, validation, categories, registration, utils, templates, the children / node helpers, and constants. Also defines + locks privateApis .
- **src/api/factory.ts ★ CORE** — Purpose: Creates and manipulates Block objects: createBlock , createBlocksFromInnerBlocksTemplate , cloneBlock , cloneSanitizedBlock , and the transform engine ( switchToBlockType , getBlockTransforms , getPossibleBlockTransformations , findTransform , isWildcardBlockTransform , isContainerGroupBlock , getBlockFromExample ).
- **src/api/registration.ts ★ CORE** — Purpose: The registration API: registerBlockType , many lookup getters ( getBlockType(s) , getBlockSupport , hasBlockSupport , ...), handler-name settings, block styles, block variations, block bindings sources, collections, and server-side bootstrapping.
- **src/api/serializer.tsx ★ CORE** — Purpose: The serializer. Converts blocks back into HTML with comment delimiters: serialize , serializeBlock , getSaveElement , getSaveContent , getBlockProps , getInnerBlocksProps , serializeAttributes , getCommentAttributes , getBlockInnerHTML , getBlockDefaultClassName .
- **src/api/parser/ ★ CORE** — Purpose: The parser pipeline: HTML source → block tree. parse is the main entry (a state machine walking block boundaries and accumulating inner content); parseWithAttributeSchema / getBlockAttributes extract typed attributes from markup; convert-legacy-block , fix-custom-classname , fix-global-attribute , apply-block-deprecated-versions , and apply-built-in-validation-fixes post-process parsed blocks.
- **src/api/validation/ ★ subsystem** — Purpose: Compares a block's source HTML against its serialized output to decide if a block is valid. isValidBlockContent , validateBlock , plus a DOM-token comparison engine ( isEquivalentHTML , token equality helpers) and the blockError / blockLog logger classes invoked during development.
- **src/api/raw-handling/ ★ subsystem** — Purpose: Turns arbitrary pasted/raw content into blocks via heuristics: rawHandler (paste flow), pasteHandler (public paste entry), html-to-blocks , get-raw-transforms , normalise-blocks , markdown-converter , shortcode-converter , and dozens of small normalisers/reducers (br/div/comment/empty-paragraph removers, heading/image/list converters, Google Docs UID remover, etc.).
- **src/api/templates.ts · categories.ts · constants.ts · children.ts · node.ts · utils.ts** — Purpose: templates.ts ( doBlocksMatchTemplate , synchronizeBlocksWithTemplate ) drives initial template/editor state; categories.ts manages categories; constants.ts holds the style/elements constants; children.ts / node.ts provide legacy BlockChildren / BlockNode HTML helpers; utils.ts holds attribute sanitization, labels, icon normalization, and roles.
- **src/store/ ★ store** — Purpose: The core/blocks Redux store holds all registered block types, variations, styles, categories, collections, and handler-name settings. actions.ts provides the ADD_BLOCK_TYPES / REMOVE_BLOCK_TYPES family; selectors.ts the read API; process-block-type.ts runs the register-time block filters; private actions/selectors extend it for bindings sources, supported styles, and keyboard shortcuts.

## Function notes

### `createBlock( name, attributes?, innerBlocks?, innerContent? )` — · src/api/factory.ts:65
- Purpose / why used: Creates a Block object given a type name and attributes. Sanitizes attributes, generates a unique clientId , and marks the block isValid: true . If the type is not registered it returns a core/missing placeholder block. Passing innerContent is only honored for core/html (Custom HTML).
- When used: Everywhere a new block instance is needed — inserting blocks, templates, cloning.
- Who uses it: createBlocksFromInnerBlocksTemplate , getBlockFromExample , cloneSanitizedBlock , the block-editor's insert logic.
- Inputs: name, attributes, innerBlocks, innerContent. Output: a Block object { clientId, name, isValid, attributes, innerBlocks } .

### `createBlocksFromInnerBlocksTemplate( innerBlocksOrTemplate )` — · src/api/factory.ts:118
- Purpose / why used: Given an array of inner-blocks templates (or already-built block objects), recursively creates the whole block tree. Handles the legacy template tuple shape [ name, attributes, innerBlocks, innerContent ] .
- When used: When materializing a template into real blocks.
- Who uses it: createBlocksFromInnerBlocksTemplate (recursion), template consumers, templates.ts .
- Inputs: array of blocks or template tuples. Output: an array of Block objects.

### `cloneBlock / cloneSanitizedBlock( block, mergeAttributes?, newInnerBlocks? )` — · src/api/factory.ts:210,153
- Purpose / why used: Returns a deep copy of a block with a fresh clientId , optionally merging new attributes and/or replacing inner blocks. cloneSanitizedBlock also sanitizes attributes (and returns a core/missing block if the type is unregistered); cloneBlock does not.
- When used: Duplicating blocks (transform into copy), copy/paste, block mover clones.
- Who uses it: The block editor's duplicate/copy actions.
- Inputs: block, mergeAttributes, newInnerBlocks. Output: a cloned block.

### `switchToBlockType( blocks, name, variationName? )` — · src/api/factory.ts:552
- Purpose / why used: Converts one or more blocks into a block (or blocks) of a new type by finding and applying the appropriate to / from transform. Gives priority to to transforms, validates multi-block matches and isMatch , runs the transform function (or __experimentalConvert ), verifies valid output that actually switches type, and fires the blocks.switchToBlockType.transformedBlock filter on each result.
- When used: The core of "Convert to Blocks", block transformation UI.
- Who uses it: The block editor's transformation UI / transformBlock .
- Inputs: blocks (array or single), target name, optional variationName. Output: an array of blocks, or null if no valid transform.

### `getBlockTransforms( direction, blockTypeOrName? ) / getPossibleBlockTransformations( blocks ) / findTransform( transforms, predicate )` — · src/api/factory.ts:494,415,459
- Purpose / why used: getBlockTransforms returns the normalized to / from transforms for a block (each augmented with blockName ) — or for all blocks when no name is given. getPossibleBlockTransformations computes the union of feasible from and to targets for a selection. findTransform returns the highest-priority transform matching a predicate (using createHooks as a priority queue).
- When used: Building the transform menu and running transformations.
- Who uses it: switchToBlockType , the block inspector's transformation UI.
- Inputs: direction + optional block; blocks array; transforms + predicate. Output: normalized transforms / candidate block types / a transform (or null).

### `getBlockFromExample( name, example )` — · src/api/factory.ts:696
- Purpose / why used: Creates a block from the block's example API definition (recursively building inner blocks from the example). Used to render block previews in the inserter.
- When used: Block previews and the block variations example rendering.
- Who uses it: The block editor's inserter previews.
- Inputs: block name + example object. Output: a Block .

### `registerBlockType / unregisterBlockType` — · src/api/registration.ts
- Purpose / why used: registerBlockType validates a block type's name/settings and dispatches addBlockTypes to the core/blocks store (the store runs processBlockType filters — blocks.registerBlockType , blocks.getBlockAttributes , blocks.registerBlockType.selectToolControls , etc.). It returns the settings or a rejection notice. unregisterBlockType dispatches removeBlockTypes .
- When used: Every block is registered this way (including core blocks).
- Who uses it: All block definitions (core + third-party).
- Inputs: block name + settings object. Output: the settings (or false on invalid).

### `getBlockType( name ) / getBlockTypes() / getBlockSupport( name, name2, default? ) / hasBlockSupport( name, ... )` — · src/api/registration.ts
- Purpose / why used: Read-only lookups into the store: a single block type, all block types, or a specific block supports value (with default fallback). hasBlockSupport is the boolean form.
- When used: Throughout the editor to inspect block metadata.
- Who uses it: getBlockTypes → transforms/logging; getBlockSupport → Block Supports API consumers.
- Inputs: name (+ support key). Output: a block type / array / support value / boolean.

### `registerBlockVariation / unregisterBlockVariation / getBlockVariations` — · src/api/registration.ts
- Purpose / why used: Register/remove/read block variations (a named variant of a block with preset attributes, e.g. "Two columns" vs "Three columns").
- When used: Block variations in the inserter.
- Who uses it: Block definitions; the block-editor's variation handling.
- Inputs: block name + variation (or variation name). Output: the variation / action.

### `registerBlockStyle / unregisterBlockStyle / registerBlockCollection / registerBlockBindingsSource (family)` — · src/api/registration.ts
- Purpose / why used: Register/remove block-specific styles (named style variations), block collections (inserter grouping), and block bindings sources (custom property bindings). Reading getters: getBlockBindingsSource(s) .
- When used: Style variants, collections in the inserter, attribute bindings.
- Who uses it: Core + third-party block authors.
- Inputs: block name + style/variation/source. Output: action objects / sources.

### `Handler-name settings + bootstrapping` — · src/api/registration.ts
- Purpose / why used: set/getFreeformContentHandlerName , set/getUnregisteredTypeHandlerName , set/getDefaultBlockName , set/getGroupingBlockName , and unstable__bootstrapServerSideBlockDefinitions (which loads server-side block.json metadata into the store).
- When used: On block bootstrap and by the parser/serializer to choose fallback handlers.
- Who uses it: The editor bootstrap and parser.
- Inputs: names. Output: void (setters) or names (getters).

### `parse( content )` — · src/api/parser/index.ts (default export)
- Purpose / why used: Parses a post's serialized HTML into an array of Block objects. It walks the markup with a state machine that recognizes comment delimiters ( <!-- wp:name /--> ), recurses into nested blocks, accumulates inner content, extracts attributes from the comment JSON, and runs the validation-fix and deprecation pipeline on each parsed block.
- When used: On post load, on block-tree hydratation.
- Who uses it: The editor's __unstableParseContent / post load flow.
- Inputs: the HTML string. Output: an array of Block objects.

### `parseWithAttributeSchema( html, schema ) / getBlockAttributes( html, blockType )` — · src/api/parser/get-block-attributes.ts
- Purpose / why used: Given a block's saved HTML and the attribute type definitions from block.json (source selectors like attribute , text , html , query , etc.), extract a typed attributes object. getBlockAttributes wraps this with the blocks.getBlockAttributes filter.
- When used: During parsing, to restore block attributes from markup.
- Who uses it: parse , and by consumers needing attribute extraction.
- Inputs: HTML + attribute schema / block type. Output: attributes object.

### `convert-legacy-block / fix-custom-classname / fix-global-attribute / apply-block-deprecated-versions / apply-built-in-validation-fixes` — · src/api/parser/ (support files)
- Purpose / why used: Post-parse normalization. applyBlockDeprecatedVersions tries each deprecated save definition to recover a block whose current save doesn't match; built-in/versioned "validation fixes" correct common historical issues; fix-custom-classname strips the auto-generated wp-block-* class when it's not a real attribute; fix-global-attribute handles the className global attribute; convert-legacy-block migrates old block names/attributes.
- When used: Inside the parser for each parsed block.
- Who uses it: parse .
- Inputs: a parsed block + parsed HTML. Output: a (possibly fixed) block.

### `serialize( blocks )` — · src/api/serializer.tsx (default export)
- Purpose / why used: Serializes an array of blocks into an HTML string, recursively handling inner blocks and wrapping each block in its comment delimiters ( <!-- wp:name ... --> ).
- When used: On save, on block-tree → string conversions.
- Who uses it: The editor's block serialization / save flow.
- Inputs: blocks array. Output: an HTML string.

### `serializeBlock( block ) / getCommentDelimitedContent( blockName, blockAttributes, innerHTML )` — · src/api/serializer.tsx
- Purpose / why used: Serializes a single block: saves its inner HTML (via the block's save function / getSaveElement / getSaveContent ), recursively serializes inner blocks, and wraps everything in the comment delimiters with serialized attributes JSON.
- When used: Inside serialize .
- Who uses it: serialize , __unstableSerializeAndClean .
- Inputs: a block. Output: the delimited HTML string.

### `getSaveElement / getSaveContent / getBlockProps / getInnerBlocksProps` — · src/api/serializer.tsx
- Purpose / why used: The save-side React bindings exposed to block authors: getSaveElement runs the block's save function in React to a ReactElement (with BlockListProvider context); getSaveContent renders that element to HTML string; getBlockProps / getInnerBlocksProps are the internal props helpers (exposed as __unstableGetBlockProps ).
- When used: During serialization and in block save / edit props.
- Who uses it: serializeBlock (getSaveElement/getSaveContent); block authors ( useBlockProps wraps these).
- Inputs: block + props/context. Output: ReactElement / HTML string / props object.

### `serializeAttributes / getCommentAttributes / getBlockInnerHTML / getBlockDefaultClassName` — · src/api/serializer.tsx
- Purpose / why used: Helpers for building the delimiter content: serializeAttributes serializes the block attribute object to a JSON-like string (with a {"key":value} / bare shape); getCommentAttributes selects which attributes go in the comment; getBlockInnerHTML reconstructs inner HTML from innerContent + inner blocks; getBlockDefaultClassName returns the wp-block-<name> class.
- When used: Inside serializeBlock .
- Who uses it: serializeBlock , consumers wanting the class name.
- Inputs: attributes / block / name. Output: strings.

### `isValidBlockContent( blockType, attributes, originalBlockContent )` — · src/api/validation/index.ts
- Purpose / why used: The top-level validation check. Serializes the block type's save output for the given attributes and compares it (semantically) with the original source markup via isEquivalentHTML . Returns true if they match.
- When used: During parsing to decide if a block is valid.
- Who uses it: validateBlock and the parser.
- Inputs: blockType, attributes, originalBlockContent. Output: boolean.

### `validateBlock( block, blockType ) / isEquivalentHTML( ... )` — · src/api/validation/index.ts
- Purpose / why used: validateBlock runs validation on a parsed block, trying isValidBlockContent against the current and (via the parser's deprecation pipeline) deprecated definitions, logging blockError s in development and producing validation messages. isEquivalentHTML is the DOM-token comparison engine (compares element structure, text content, attributes, and style with normalization and equality helpers).
- When used: During parsing; validation messages surface in the UI.
- Who uses it: The parser and block-invalid UI.
- Inputs: a block + block type / two HTML strings. Output: validation result / boolean.

### `rawHandler( { HTML } ) / pasteHandler( { HTML, ... } )` — · src/api/raw-handling/
- Purpose / why used: Convert arbitrary HTML (from paste or raw input) into an array of blocks. rawHandler applies the get-raw-transforms pipeline (a series of normalisers then transformers then blockParsers ); pasteHandler is the public paste entry that detects file/image/shortcode/markdown/HTML and routes accordingly, then falls back to html-to-blocks .
- When used: On paste events and raw/HTML insertion.
- Who uses it: The block editor's paste/insert handlers.
- Inputs: the raw HTML (+ paste config). Output: an array of Block s.

### `get-raw-transforms / normalise-blocks / html-to-blocks + the normalisers/reducers` — · src/api/raw-handling/
- Purpose / why used: The raw-handling pipeline pieces. get-raw-transforms returns the ordered list of raw transforms ( blockquote , heading , hr , list , pre , p , table ); the individual files are DOM normalisers (remove specific nodes/attrs, correct figures, expand abbreviations) and converters (markdown, shortcode, MS list, Slack, LaTeX). html-to-blocks runs the final normalise-blocks /block-parse step; is-inline-content decides whether pasted content is paragraph-only.
- When used: Inside rawHandler / pasteHandler .
- Who uses it: rawHandler , pasteHandler .
- Inputs: HTML/filters. Output: normalized HTML / transforms list / blocks.

### `store (register)` — · src/store/index.ts:15
- Purpose / why used: Creates and registers the core/blocks Redux store, wiring its reducer, selectors, actions, and private selectors/actions.
- When used: On module load.
- Who uses it: All components via @wordpress/data ; the registration API dispatches here.
- Inputs: reducer + selectors + actions. Output: the registered store.

### `addBlockTypes / removeBlockTypes / reapplyBlockTypeFilters` — · src/store/actions.ts:22,85,43
- Purpose / why used: Add/remove block types in state. reapplyBlockTypeFilters is a thunk that re-processes all stored unprocessed block types through processBlockType ( blocks.registerBlockType filters) so late-registered filters still apply to all blocks.
- When used: From registerBlockType / unregisterBlockType ; reapplyBlockTypeFilters on filter-hook updates.
- Who uses it: The registration API; the block filter system.
- Inputs: block type(s) / names. Output: { type: 'ADD_BLOCK_TYPES', blockTypes } etc.

### `addBlockStyles / removeBlockStyles / addBlockVariations / removeBlockVariations` — · src/store/actions.ts:103,125,147,169
- Purpose / why used: Add/remove named styles and variations for blocks.
- When used: From registerBlockStyle / registerBlockVariation .
- Who uses it: The registration API.
- Inputs: block name + styles/variations. Output: action objects.

### `setDefaultBlockName / setFreeformFallbackBlockName / setUnregisteredFallbackBlockName / setGroupingBlockName / setCategories / updateCategory / addBlockCollection / removeBlockCollection` — · src/store/actions.ts:192-333
- Purpose / why used: The remaining state setters: default/fallback/grouping handler block names, categories (set + update one), and block collections.
- When used: From the corresponding registration API functions.
- Who uses it: The registration API; bootstrap.
- Inputs: names / categories. Output: action objects.

### `getBlockTypes / getBlockType / getBlockStyles / getBlockVariations` — · src/store/selectors.ts
- Purpose / why used: The main read API. getBlockTypes returns all block types (memoized over the blockTypes slice); getBlockType returns one by name; getBlockStyles and getBlockVariations return a block's styles/variations (with variation defaulting, scoped by an optional scope ).
- When used: Throughout the editor and the registration API.
- Who uses it: All block inspection code.
- Inputs: state (+ name / scope). Output: block types / variations / styles.

### `getBlockSupport / hasBlockSupport / getCategories / getDefaultBlockName (family)` — · src/store/selectors.ts
- Purpose / why used: Read specific metadata: a block's support value (with default), hasBlockSupport boolean, the list of categories, and the handler-name constants ( getDefaultBlockName , getFreeformFallbackBlockName , getUnregisteredFallbackBlockName , getGroupingBlockName ).
- When used: Everywhere block metadata is read.
- Who uses it: The Block Supports API, registration getters, the inserter.
- Inputs: state (+ block name / support key). Output: support value / boolean / strings.

### `private selectors: getSupportedStyles / getUnprocessedBlockTypes / getBlockBindingsSource(s) / getBlockKeyboardShortcuts` — · src/store/private-selectors.ts
- Purpose / why used: Private read API for core internal use: supported style properties, unprocessed (pre-filter) block types (used by reapplyBlockTypeFilters ), block bindings sources, keyboard shortcuts, and content-role attribute checks.
- When used: From core bootstrapping / private consumers via unlock .
- Who uses it: Core editor modules.
- Inputs: state (+ args). Output: varieds.

### `processBlockType (applyBlockFilters)` — · src/store/process-block-type.ts
- Purpose / why used: The register-time filter processing function (dispatched as a thunk by addBlockTypes ). Applies the blocks.registerBlockType and blocks.getBlockAttributes filters to a block's settings/attributes, runs the blocks.registerBlockType.selectToolControls filter, and normalizes the result — including computing the block.json attributes-to-roles mapping.
- When used: On every block registration.
- Who uses it: addBlockTypes (via reapplyBlockTypeFilters and direct dispatch).
- Inputs: name + settings. Output: the processed block type (or undefined).


---

# packages/rich-text — full file & function map

**Owns:**
- The RichTextValue data model ( text + parallel formats / replacements arrays + selection).
- The pure value-transform functions that are the building blocks of the RichText block.
- The DOM ↔ value round-trip ( create , toDom / apply , toHTMLString ).
- The format-type registration store ( core/rich-text ).
- The useRichText hook and the event-listener set that keeps the model in sync with a live editable DOM.

## File map / file details

- **src/index.ts ★ ENTRY** — Purpose: The package entry point. Re-exports all public symbols: the store, all the transform functions ( create , createElement , concat , slice , split , join , insert , insertObject , replace , remove , applyFormat , removeFormat , toggleFormat , updateFormats (via privateApis ), ...), the format lookups, the serializers ( toHTMLString , toDom ), the format registration functions, the store helpers, the hooks ( useAnchorRef , useAnchor , __unstableUseRichText ), the RichTextData class, types, and privateApis .
- **src/private-apis.js** — Purpose: Assembles and locks the private API object only Gutenberg core modules may consume: useRichText , KeyboardShortcutContext , InputEventContext , RichTextShortcut , RichTextInputEvent , shortcutsListener , inputEventsListener , ownsSelection , subscribeOwnedListener .
- **src/types.ts · src/special-characters.js** — Purpose: types.ts defines the core type shapes ( RichTextFormat , RichTextFormatList , RichTextValue ); special-characters.js defines the two sentinel characters ( OBJECT_REPLACEMENT_CHARACTER = \ufffc and ZWNBSP = \ufeff ).
- **src/create.js ★ CORE** — Purpose: The main entry for creating a RichTextValue from HTML, DOM, or plain text. Recursively walks a DOM element, accumulating text, formats, replacements, and selection offsets. Also defines the RichTextData wrapper class.
- **src/to-tree.js · src/to-dom.js · src/to-html-string.js ★ CORE** — Purpose: The serialization stack. toTree converts a value to an abstract tree (shared core algorithm); toDom renders it to a live DOM tree and apply diffs it onto an editable element; toHTMLString renders the tree to an HTML string.
- **src/concat · slice · split · join · insert · insert-object · replace · remove · remove-format · apply-format · toggle-format · update-formats · normalise-formats** — Purpose: The pure-functional value-transform API. Each mirrors a String / Array operation but carries the parallel formats / replacements arrays and selection, normalizing the result.
- **src/get-active-format · get-active-formats · get-active-object · get-format-type(s) · get-text-content · is-collapsed · is-empty · is-format-equal · is-range-equal** — Purpose: Read-only lookups and predicates. getActiveFormat(s) determine which formats apply across the selection; getActiveObject detects selection of exactly one object; getFormatType(s) read the store; the is-* helpers are equality / state checks.
- **src/register-format-type · unregister-format-type** — Purpose: Registers and unregisters a format type with validation, dispatching addFormatTypes / removeFormatTypes to the core/rich-text store.
- **src/owns-selection.js · src/subscribe-owned-listener.js** — Purpose: DOM selection utilities. ownsSelection checks whether an element owns the current document selection; subscribeOwnedListener delegates events that fire only when a given element owns the selection.
- **src/keyboard-shortcut.js · input-event.js · contexts.js · event-listeners.ts** — Purpose: The format-type ↔ editable plumbing. RichTextShortcut and RichTextInputEvent register callbacks into the KeyboardShortcutContext / InputEventContext reft Sets; event-listeners.ts provides the factories that attach the matching DOM listeners and invoke those callbacks.
- **src/store/index.js · actions.js · reducer.js · selectors.js** — Purpose: Defines and registers the core/rich-text Redux store. State is a record of registered format types. Actions: addFormatTypes , removeFormatTypes . Selectors: getFormatTypes , getFormatType , getFormatTypeForBareElement , getFormatTypeForClassName .
- **src/hook/index.js ★ CORE** — Purpose: useRichTextBase (the raw hook) and useRichText (the format-type-aware wrapper). Bridges the value model to a live contentEditable via refs, create , toDom 's apply , and toHTMLString . Also exports useDeprecatedRichText (as __unstableUseRichText ).
- **src/hook/use-anchor.ts · use-anchor-ref.js** — Purpose: Return the active format element, or a virtual anchor over the selection range, for positioning a Popover (e.g. the link toolbar). useAnchorRef is the deprecated predecessor.
- **src/hook/use-boundary-style.js · use-default-style.js · use-format-types.js** — Purpose: Supporting hooks. useBoundaryStyle injects a highlight CSS rule for the deepest active format; useDefaultStyle forces white-space: pre-wrap ; useFormatTypes fetches registered format types and derives their prepare/value/change handlers.
- **src/hook/event-listeners/ ★ subsystem** — Purpose: The DOM listener set that keeps the internal record synced with the live editable. index.js composes them in a fixed order; input-and-selection.js is the most complex (input, selection, composition, focus sync); the rest handle copy/cut, full-content delete, arrow-key format navigation, object click selection, focus-capture and cross-browser selectionchange .

## Function notes

### `create( options? )` — · src/create.js
- Purpose / why used: Create a RichTextValue from an HTML string, a DOM element, a range, or plain text.
- When used: On mount to parse initial content, and on every DOM → model sync ( createRecord in the hook reads the live selection).
- Who uses it: RichTextData , useRichTextBase , insert , join , all string-wrapping call sites.
- Inputs: { element?, text?, html?, range?, __unstableIsEditableTree? } . Output: a RichTextValue .

### `class RichTextData` — · src/create.js
- Purpose / why used: A string-like wrapper around a RichTextValue with convenience factories and methods. Proxies String.prototype methods through toHTMLString() so it behaves like a string in most contexts.
- When used: As the value type for RichText content (block attribute values).
- Who uses it: The block editor's RichText component and any consumer wanting a richer content value than a plain string.
- Inputs: none (factories take text/html/element). Output: a RichTextData .

### `concat( ...values ) / mergePair( a, b )` — · src/concat.js
- Purpose / why used: Merge multiple RichTextValue s into one (appends formats, replacements, and text). mergePair mutates a ; concat reduces and normalizes.
- When used: Joining split fragments; createFromElement uses mergePair to accumulate text nodes.
- Who uses it: create.js (mergePair), join.js , consumers.
- Inputs: one or more values. Output: a normalized combined value.

### `insert( value, valueToInsert, startIndex?, endIndex? ) / insertObject( value, format, ... )` — · src/insert.js · insert-object.js
- Purpose / why used: Insert content into a range (removing anything in between), moving the caret after the insertion. insertObject inserts a non-text object (e.g. an image) as a formatted replacement.
- When used: Whenever text or an inline object is typed/pasted/inserted.
- Who uses it: The RichText component, inline image/embed insertion.
- Inputs: value, value-to-insert (or format), start/end indices. Output: the new value.

### `applyFormat( value, format, startIndex?, endIndex? )` — · src/apply-format.js
- Purpose / why used: Apply (add) a format to a range. For a collapsed caret inside an existing format of the same type, it walks outward and updates that format (allows attribute editing); for a non-collapsed selection it splices the format in at the shallowest nesting depth, and updates activeFormats .
- When used: When the user toggles bold/italic/link styles.
- Who uses it: toggleFormat , format toolbar buttons.
- Inputs: value, format, indices. Output: the new value.

### `removeFormat( value, formatType, startIndex?, endIndex? )` — · src/remove-format.js
- Purpose / why used: Remove a format type from a range. For a collapsed caret it expands to the whole contiguous run of that format object and removes it; for a selection it filters the type from each index.
- When used: To clear a style.
- Who uses it: toggleFormat , "clear formatting" controls.
- Inputs: value, formatType, indices. Output: the new value.

### `toggleFormat( value, format )` — · src/toggle-format.js
- Purpose / why used: Toggle a format at the current selection, announcing the action via the @wordpress/a11y speak() helper.
- When used: Format toolbar buttons and keyboard shortcuts.
- Who uses it: The RichText toolbar / shortcuts.
- Inputs: value, format. Output: the toggled value.

### `slice / split / join / remove / replace` — · src/slice.js · split.js · join.js · remove.js · replace.js
- Purpose / why used: The remaining String -analog transforms: slice extracts a sub-range; split splits at a delimiter string or the selection range; join joins an array with a separator; remove deletes a range (inserts an empty value); replace does search-and-replace (accepting a function, string, or value as replacement).
- When used: Throughout the editor for text manipulation.
- Who uses it: The RichText component, paste handling ( replace ), formatting operations.
- Inputs: value + indices / delimiters / pattern. Output: new value(s).

### `updateFormats( { value, start, end, formats } )` — · src/update-formats.js
- Purpose / why used: Efficiently set active formats across a range after input; reuses reference identities for stability and sets activeFormats .
- When used: In input-and-selection.js right after a text insertion, to apply formats to the newly typed characters.
- Who uses it: The input-and-selection listener.
- Inputs: a value + start/end + formats. Output: the (mutated) value.

### `normaliseFormats( value )` — · src/normalise-formats.js
- Purpose / why used: Ensure adjacent identical formats share the same object reference (critical for DOM diffing performance and selection stability).
- When used: At the end of every mutating transform.
- Who uses it: applyFormat , removeFormat , insert , concat , join , replace .
- Inputs: value. Output: a new value with deduplicated format references.

### `toTree( options )` — · src/to-tree.js
- Purpose / why used: The core algorithm converting a RichTextValue into an abstract tree. It walks every character position, builds a stacked-object structure from the format stack, renders object replacements, <br> for newlines, text nodes, selection markers, and ZWNBSP padding. Used by both toDom (live DOM) and toHTMLString (HTML).
- When used: On every serialization.
- Who uses it: to-dom.js and to-html-string.js .
- Inputs: value + a set of tree-operation callbacks ( createEmpty , append , appendText , etc.). Output: a tree node.

### `toDom( { value, ... } ) → { body, selection }` — · src/to-dom.js
- Purpose / why used: Convert a value to a live DOM tree and return it plus selection paths (index arrays from root to the selection nodes). Handles the MathML namespace for <math> .
- When used: Before applying to the editable, and to slice the selected HTML.
- Who uses it: apply , consumers needing the DOM representation.
- Inputs: value (+ prepareEditableTree , isEditableTree , placeholder , doc ). Output: { body, selection } .

### `apply( { value, current, ... } )` — · src/to-dom.js
- Purpose / why used: Build a new tree via toDom and diff it against the live editable element ( applyValue ), then set the selection ( applySelection ) unless __unstableDomOnly .
- When used: In useRichTextBase.applyRecord to sync the DOM to a new value.
- Who uses it: The useRichText hook.
- Inputs: value + current element. Output: none (DOM side effects).

### `toHTMLString( { value, preserveWhiteSpace? } )` — · src/to-html-string.js
- Purpose / why used: Serialize a value to an HTML string using plain-object tree operations (no DOM), recursively rendering text (escaped), elements (with escaped attributes), comments, and self-closing objects.
- When used: When the block needs its HTML form (save, paste, API).
- Who uses it: RichTextData , the RichText change handler, consumers.
- Inputs: value + preserveWhiteSpace . Output: an HTML string.

### `addFormatTypes / removeFormatTypes` — · src/store/actions.js
- Purpose / why used: Add or remove registered format types in state (normalizing to arrays).
- When used: From registerFormatType / unregisterFormatType .
- Who uses it: The registration functions; format-type plugins.
- Inputs: formatTypes / names. Output: action objects ( { type: 'ADD_FORMAT_TYPES', formatTypes } / { type: 'REMOVE_FORMAT_TYPES', names } ).

### `getFormatType / getFormatTypes / getFormatTypeForBareElement / getFormatTypeForClassName` — · src/store/selectors.js
- Purpose / why used: Read registered format types: all, by name, by bare tag name (or wildcard '*' ), or by class name.
- When used: getFormatType(s) in root lookups; tag/class lookups during DOM parsing ( create.js 's toFormat ).
- Who uses it: get-format-type.js , get-format-types.js , create.js , to-tree.js , use-format-types.js .
- Inputs: state (+ name/tagName/className). Output: a format type or an array.

### `useRichTextBase( props ) / useRichText( props )` — · src/hook/index.js
- Purpose / why used: The main hook powering RichText components: bridges the value model with a live contentEditable DOM element. useRichTextBase creates records from the DOM ( createRecord ), applies values back ( applyRecord via apply ), handles change serialization, and merges all the supporting hook refs ( useDefaultStyle , useBoundaryStyle , useEventListeners ). useRichText adds format-type integration ( useFormatTypes ) and the editor-only format handlers.
- When used: Inside the RichText component (via private APIs).
- Who uses it: The block editor's RichText component.
- Inputs: value, selection, placeholder, onChange, format options, etc. Output: { value, getValue, onChange, ref, formatTypes } .

### `useAnchor( { editableContentElement, settings? } )` — · src/hook/use-anchor.ts
- Purpose / why used: Returns the active format element or a virtual anchor (with a getBoundingClientRect over the selection range) for positioning a Popover. Subscribes to selectionchange to stay current.
- When used: By Popovers bound to a format (link toolbar, color picker).
- Who uses it: The block editor's formatting UI.
- Inputs: the editable element + format settings. Output: an element or a virtual anchor (or null).

### `useFormatTypes( { allowedFormats, withoutInteractiveFormatting, ... } )` — · src/hook/use-format-types.js
- Purpose / why used: Fetches registered format types, filters by allowedFormats and the interactive-format exclusion, and builds the prepareHandlers , valueHandlers , and changeHandlers from the experimental format-type hooks.
- When used: Inside useRichText .
- Who uses it: useRichText .
- Inputs: allowedFormats, withoutInteractiveFormatting, context. Output: { formatTypes, prepareHandlers, valueHandlers, changeHandlers, dependencies } .

### `useEventListeners( props ) — the composed listener set` — · src/hook/event-listeners/index.js
- Purpose / why used: Composes all the per-module event listener hooks into a single ref effect, in a fixed order, using useInsertionEffect to keep a fresh propsRef without re-running ref effects every render.
- When used: Inside useRichTextBase .
- Who uses it: useRichTextBase .
- Inputs: the hook props. Output: a ref effect (plus cleanups).

### `input-and-selection listener (the most complex)` — · src/hook/event-listeners/input-and-selection.js
- Purpose / why used: Synchronizes the internal record with the live DOM. Handles input (recreate record + updateFormats for new chars), selectionchange (ownership check, empty-field caret fix, active-format update, onSelectionChange ), compositionstart/end , focusin (nested editable handling), and early capture-phase sync on keydown/beforeinput/copy/cut/paste.
- When used: On every editable interaction.
- Who uses it: The composed listener set.
- Inputs: element + propsRef. Output: cleanup (listener removal).

### `format-boundaries / select-object / copy-handler / delete / prevent-focus-capture / selection-change-compat` — · src/hook/event-listeners/
- Purpose / why used: The remaining specialized listeners: format-boundaries manages activeFormats on left/right arrow navigation; select-object selects clicked inline objects; copy-handler serializes selection to text/plain + text/html on copy/cut (and removes on cut); delete handles full-content delete; prevent-focus-capture fixes flex-parent focus capture; selection-change-compat fires a synthetic selectionchange for browsers that miss it after mouse/keyboard changes.
- When used: On the corresponding DOM events.
- Who uses it: The composed listener set.
- Inputs: element + propsRef. Output: cleanup functions.


---

# packages/core-data - full file & function map

**Owns:**
- The entity registry : rootEntitiesConfig + lazy additionalEntityConfigLoaders (post types, taxonomies, site) that define how any REST resource is treated as an entity.
- The normalized queried-data cache ( queried-data/ ) - items stored by context+id, indexed by query, with pagination and field filtering.
- The canonical CRUD selectors/actions/resolvers ( getEntityRecord(s) , editEntityRecord , saveEntityRecord , deleteEntityRecord , saveEditedEntityRecord ).
- Undo/redo via the @wordpress/undo-manager , and an in-memory locking engine ( locks/ ) used by the editor's save flow.
- Permissions ( canUser via OPTIONS), autosaves/revisions , and the user/theme/pattern sub-caches.
- The React hooks ( useEntityRecord , useEntityRecords , useEntityProp , ...) and the EntityProvider React context.
- The collaboration (CRDT/Yjs sync) subsystem and the EntitiesSavedStates save panel (both gated).

## File map / file details

- **src/index.js** — Purpose: The package entry point. Builds the core store config (merging static + dynamic entity actions/selectors/resolvers), creates and registers the store via createReduxStore( STORE_NAME, storeConfig() ) , registers private selectors/actions, and re-exports EntityProvider , the entity-types, the fetch utilities, the hooks, and the locked private APIs.
- **src/name.js** — Purpose: Single export: export const STORE_NAME = 'core' . The one store this package owns. (Other stores like core/edit-widgets are registered by their own packages.)
- **src/entities.js** — Purpose: Defines rootEntitiesConfig (21 static entity definitions) and additionalEntityConfigLoaders (lazy loaders for post types, taxonomies, and the site entity). This is the source of truth for how every REST resource maps to an entity (kind, name, baseURL, key, labels, rawAttributes, sync config).
- **src/selectors.ts** — Purpose: The canonical read API. Selectors for entity records, records-by-query, edits, save/delete status, dirty entities, undo/redo, current user/theme, block patterns, embeds, permissions, autosaves, revisions, and more. Includes the memoized and registry selectors.
- **src/actions.js** — Purpose: The canonical write API. Actions for editing, saving, deleting, receiving entity records, undo/redo, batch requests, and various receive* hydration actions.
- **src/resolvers.js** — Purpose: Data resolvers - thunks that fetch REST data and dispatch receive actions, with shouldInvalidate hints for cache invalidation. Includes the core getEntityRecord / getEntityRecords resolvers and many entity-specific ones.
- **src/reducer.js** — Purpose: The root reducer. Builds a dynamic per-entity sub-reducer tree plus slices for users, current user/theme, embed previews, user permissions, autosaves, block patterns, undo manager, and the sync state.
- **src/private-selectors.ts / private-actions.js / private-apis.ts** — Purpose: The locked private surface. Selectors/actions reachable only via unlock( store ) (registered in index.js ). Used by other editor packages for things like getEditorSettings , getHomePage , saveDirtyEntities , and collaboration state.
- **src/sync.ts** — Purpose: Lazily wraps @wordpress/sync : getSyncManager() , hasSyncManager() , and CRDT snapshot helpers ( getEntitySnapshot , entityContainsSnapshot ). Only activates when the real-time-collaboration experiment is enabled.
- **src/entity-provider.jsx / entity-context.js** — Purpose: EntityProvider provides an EntityContext (kind/name/id) to its subtree, and useEntityId / useEntityProp read from it.
- **src/queried-data/actions.js** — Purpose: Actions receiveItems , removeItems , and receiveQueriedItems that feed the normalized store.
- **src/queried-data/reducer.js** — Purpose: Combines items (by context then id), itemIsComplete (tracks whether all fields were received), and queries (stable query key to { itemIds, meta } ). Uses getMergedItemIds for pagination merging. Handles RECEIVE_ITEMS / REMOVE_ITEMS .
- **src/queried-data/selectors.js** — Purpose: getQueriedItems (with WeakMap caching + deep-query equality), getQueriedTotalItems , getQueriedTotalPages . Supports field filtering, pagination, and include.
- **src/queried-data/get-query-parts.js** — Purpose: Parses a query object into a stable cache key and pagination parts ( { stableKey, page, perPage, offset, include, fields, context } ).
- **src/hooks/use-entity-record.ts** — Purpose: useEntityRecord(kind, name, id, options?) - wraps useSelect to return { record, editedRecord, hasEdits, ... } .
- **src/hooks/use-entity-records.ts** — Purpose: useEntityRecords(kind, name, query?, options?) - wraps useSelect to return a query result set.
- **src/hooks/use-entity-prop.js · use-entity-id.js · use-entity-block-editor.js · use-resource-permissions.ts · use-query-select.ts** — Purpose: useEntityProp (read/write one prop of an entity), useEntityId (read id from context), useEntityBlockEditor ( [blocks, onInput, onChange] for block editing an entity), useResourcePermissions (canCreate/Read/Update/Delete), useQuerySelect (resolveSelect-aware hook).
- **src/locks/** — Purpose: An in-memory optimistic locking engine ( createLocks : acquire(store, path, { exclusive }) / release ) used by entity resolvers/actions to coordinate concurrent access. Wrapped as __unstableAcquireStoreLock / __unstableReleaseStoreLock actions.
- **src/batch/** — Purpose: createBatch() - coalesces many REST requests and sends them in a single multipart request via defaultProcessor to __experimental/batch . Exposed as the __experimentalBatch action.
- **src/fetch/** — Purpose: Standalone fetch helpers: __experimentalFetchLinkSuggestions (link search), __experimentalFetchUrlData (embed data), and fetchBlockPatterns .
- **src/components/entities-saved-states/** — Purpose: The "Save" panel in the site editor, grouping dirty entity records by type with per-record toggles. Exposed via the locked private APIs as EntitiesSavedStates / EntitiesSavedStatesExtensible .
- **src/footnotes/** — Purpose: updateFootnotesFromMeta(blocks, meta) - reorders footnote markup across blocks and keeps ordered footnote ids in sync with post meta.
- **src/awareness/** — Purpose: Collaborative editing awareness (cursor positions, block selections) used by the real-time collaboration experiment.
- **src/entity-types/** — Purpose: TypeScript type definitions for every entity record ( EntityRecord<C> , Updatable<T> , Context ) and each concrete entity (Post, Page, Template, User, ...).
- **src/utils/** — Purpose: ~28 small helpers: query/item processing ( conservativeMapItem , getFilteredItem , getPaginationMeta ), reducer HOCs ( ifMatchingAction , replaceAction ), memoization ( withWeakMapCache ), permissions ( user-permissions.js ), entity shortcuts ( forward-resolver.js , log-entity-deprecation.ts ), and the CRDT collaboration utilities.

## Function notes

### `storeConfig()` — - src/index.js:103
- Purpose / why used: Assembles the core store definition by merging the static and dynamically-generated entity actions/selectors/resolvers into one config object.
- When used: Once, at module load, inside createReduxStore(STORE_NAME, storeConfig()) .
- Who uses it: index.js itself.
- Inputs: none (reads rootEntitiesConfig and additionalEntityConfigLoaders ).
- Output: the store config { reducer, actions, selectors, resolvers } .

### `store (register)` — - src/index.js:124
- Purpose / why used: Creates and registers the core Redux store, then registers the private selectors and actions.
- When used: On module load (importing @wordpress/core-data ).
- Who uses it: Every editor package that reads core data.
- Inputs: storeConfig() . Output: the registered store (exported as store ).

### `Dynamic entity shortcuts (build-time)` — - src/index.js:23-101
- Purpose / why used: For every entity in rootEntitiesConfig , generates ergonomic shortcut selectors/resolvers/actions like getGlobalStyles , savePost , deletePage that forward to the canonical getEntityRecord(s) / saveEntityRecord / deleteEntityRecord and log a deprecation notice ( log-entity-deprecation.ts ).
- When used: Applied when the store config actions/selectors/resolvers are assembled.
- Who uses it: The store config ( storeConfig() ); consumers of the generated method names.
- Inputs: entity config (kind, name, key). Output: generated thunk + selector objects.

### `rootEntitiesConfig` — - src/entities.js:27-245
- Purpose / why used: The static array of 21 root entity definitions. Each maps an entity to a REST base URL, defines its key field, plural, label, and whether it supports pagination, raw attributes, transient/merged edits, and sync config.
- When used: At store-config assembly (to build shortcuts) and by resolvers (to know how to fetch).
- Who uses it: index.js , getEntitiesConfig resolver, entity selectors/actions.
- Inputs: none. Output: the config array.

### `additionalEntityConfigLoaders` — - src/entities.js:259-268
- Purpose / why used: Lazy entity-config loaders fetched on demand by the getEntitiesConfig resolver, because post-type/taxonomy/site entities depend on runtime API data.
- When used: First time an entity of a dynamic kind is requested.
- Who uses it: The getEntitiesConfig resolver in resolvers.js .
- Inputs: none. Output: loader functions that return entity configs.

### `loadPostTypeEntities() / loadTaxonomyEntities() / loadSiteEntity()` — - src/entities.js:332,481,510
- Purpose / why used: Fetch the entity config for every registered post type (from /wp/v2/types?context=view ), every taxonomy ( /wp/v2/taxonomies ), and the single site entity (via OPTIONS /wp/v2/settings ). Post-type entities gain transientEdits (blocks + selection), mergedEdits (meta), rawAttributes (title/excerpt/content), and sync config; the site entity gets transientEdits for template-less editing.
- When used: Via the loaders on first dynamic-kind request.
- Who uses it: The getEntitiesConfig resolver.
- Inputs: none. Output: arrays of entity config objects (added to store config).

### `getMethodName( kind, name, prefix )` — - src/entities.js:557
- Purpose / why used: Generates PascalCase method names like getPostType , getPosts , savePost , deletePost from an entity's kind+name and a verb prefix, so the dynamic shortcuts can be named deterministically.
- When used: During store-config assembly in index.js .
- Who uses it: The dynamic-shortcut generator in index.js .
- Inputs: kind, name, prefix. Output: the generated method name string.

### `getEntityRecord( state, kind, name, recordId?, query? )` — - src/selectors.ts:412
- Purpose / why used: The core single-record reader. Returns the raw fetched record for an (kind, name, recordId) tuple, read from the normalized queriedData section. The query is used for incomplete-record handling (merging subsequent field fetches).
- When used: Everywhere a single entity is needed. It is what all higher packages call to read, e.g., a current post.
- Who uses it: The whole editor stack ( editor , edit-post , edit-site , block binding sources), plus useEntityRecord .
- Inputs: state, entity kind, entity name, record id, optional query. Output: the record object or undefined (which triggers its resolver).

### `getEntityRecords( state, kind, name, query )` — - src/selectors.ts:719
- Purpose / why used: The core collection reader. Returns an array of records matching a query, resolved through the queried-data query index, with pagination/field/context handling.
- When used: Whenever a list of entities is needed (e.g. all pages, posts in a category, reusable block list).
- Who uses it: useEntityRecords , block editor list components, the site editor, etc.
- Inputs: state, kind, name, query. Output: array of records (or undefined when not yet resolved).

### `getEditedEntityRecord( state, kind, name, recordId )` — - src/selectors.ts:1037
- Purpose / why used: Returns the record with local (unsaved) edits merged on top, filtering out transient edits (like block selection ) and applying mergedEdits deep-merge for fields such as meta .
- When used: Everywhere the UI edits a record and needs to reflect unsaved changes.
- Who uses it: The editor ( getEditedPostAttribute delegates here via private API), form fields, useEntityRecord ( editedRecord ).
- Inputs: state, kind, name, recordId. Output: the merged record.

### `hasEditsForEntityRecord / getEntityRecordEdits / getEntityRecordNonTransientEdits` — - src/selectors.ts:1012,942,968
- Purpose / why used: Read whether a record has unsaved edits and what they are. getEntityRecordNonTransientEdits strips transient (per-frame) edits like selection .
- When used: To decide whether to show save UI, enable the Save button, or gather the payload for a save.
- Who uses it: saveEditedEntityRecord , the site editor save panel, editor dirty-state checks.
- Inputs: state, kind, name, recordId. Output: boolean / edits object.

### `isSavingEntityRecord / isAutosavingEntityRecord / isDeletingEntityRecord / getLastEntitySaveError / getLastEntityDeleteError` — - src/selectors.ts:1113,1091,1137,1161,1183
- Purpose / why used: Read the in-flight status and last error of a record's save/delete, including autosave state.
- When used: To render spinners/disable buttons while saving and to surface save errors.
- Who uses it: Save buttons, the editor's isSavingPost / getLastSaveError via private API, the site editor save panel.
- Inputs: state, kind, name, recordId. Output: boolean / error object.

### `__experimentalGetDirtyEntityRecords / __experimentalGetEntitiesBeingSaved` — - src/selectors.ts:830,887
- Purpose / why used: Return every entity record that has unsaved edits (dirty), and every entity whose save is currently in flight.
- When used: The site editor's "Save" panel and save flow enumerate dirty entities from here.
- Who uses it: EntitiesSavedStates , saveDirtyEntities private action, the editor save button.
- Inputs: state. Output: array of record descriptors.

### `hasUndo( state ) / hasRedo( state )` — - src/selectors.ts:1240,1255
- Purpose / why used: Report whether there are undo/redo levels available, from the undo manager state.
- When used: To enable/disable the editor's undo/redo toolbar buttons.
- Who uses it: Editor toolbar, keyboard shortcuts.
- Inputs: state. Output: boolean.

### `canUser( state, action, resource, id? )` — - src/selectors.ts:1351
- Purpose / why used: Returns whether the current user can perform an action on a REST resource, from the userPermissions cache populated by an OPTIONS request (its resolver).
- When used: To conditionally enable/disable UI and hide controls the user cannot use.
- Who uses it: Editor UI (publish, delete, per-field capability checks).
- Inputs: state, action (e.g. 'create'/'update'/'delete'), resource, optional id. Output: boolean.

### `getCurrentUser / getCurrentTheme / getThemeSupports / getEmbedPreview / getBlockPatterns / getRevisions / getRevision / getDefaultTemplateId ...` — - src/selectors.ts:192-1701 (selected)
- Purpose / why used: The remaining specialized readers: the current user object and authors; the active theme + its global-styles revisions; embed previews; registered block patterns/categories and user pattern categories; post revisions ( getRevisions , getRevision , hasRevision ); and the default template id for a query. Each maps to its own cache slice and resolver.
- When used: Throughout the editor (user avatar/name, theme-dependent styles, embed block, pattern inserter, revision UI, template resolution).
- Who uses it: editor , block-editor , edit-site , embed block, pattern library.
- Inputs/Output: state (+ resource-specific args) to the cached value.

### `editEntityRecord( kind, name, recordId, edits, options )` — - src/actions.js:431
- Purpose / why used: Applies a set of edits to an entity record's local edits, records an undo level, and (in collaboration mode) reports the edits to the sync manager. The single sanctioned way to mutate entity data locally.
- When used: Every time the UI changes an entity property (title, content, meta, ...).
- Who uses it: Translation of editPost in @wordpress/editor ; binding sources; useEntityProp 's setter; all forms.
- Inputs: kind, name, recordId, partial edits object, options ( undoIgnore , etc.). Output: dispatches EDIT_ENTITY_RECORD (and side-effects for sync/undo).

### `saveEntityRecord( kind, name, record, options )` — - src/actions.js:633
- Purpose / why used: Persists an entity record to the REST API (handling autosave vs regular save, non-deterministic fields, locks, and, in collaboration mode, CRDT doc sync). Dispatches SAVE_ENTITY_RECORD_START / _FINISH / _FAILED and resolves on the saved record.
- When used: When the user saves an entity (post/page/pattern/global styles/...).
- Who uses it: saveEditedEntityRecord , the editor save flow, the site editor save panel.
- Inputs: kind, name, record (with id), options ( isAutosave , throwOnError ). Output: a promise resolving to the saved entity record (or throwing on error).

### `saveEditedEntityRecord( kind, name, recordId, options )` — - src/actions.js:970
- Purpose / why used: Gathers the record's non-transient edits and saves them, then clears the local edits on success. The recommended way to save an edited record.
- When used: The editor's save flow and site editor "Save" button.
- Who uses it: Translation of savePost / saveEditedEntityRecord in @wordpress/editor ; the site editor.
- Inputs: kind, name, recordId, options. Output: promise resolving to the saved record.

### `deleteEntityRecord( kind, name, recordId, query, options )` — - src/actions.js:336
- Purpose / why used: Deletes an entity record via DELETE , dispatching DELETE_ENTITY_RECORD_START/_FINISH/_FAILED and removing it from the cache.
- When used: When the user deletes a post/page/pattern/etc.
- Who uses it: Editor trash/delete flow, pattern library, media library.
- Inputs: kind, name, recordId, query, options. Output: promise resolving to the deleted record or error.

### `undo() / redo() / __unstableCreateUndoLevel()` — - src/actions.js:578,595,613
- Purpose / why used: undo / redo call into the @wordpress/undo-manager ; __unstableCreateUndoLevel captures an in-memory block-tree snapshot (used by rect editor edits) into an undo level.
- When used: Editor Undo/Redo toolbar and shortcuts; editEntityRecord calls createUndoLevel before each edit.
- Who uses it: Editor UI, keyboard shortcuts.
- Inputs: (undo/redo) none. Output: dispatch of UNDO / REDO , or __UNSTABLE_CREATE_UNDO_LEVEL with the snapshot.

### `receiveEntityRecords( kind, name, records, query?, ... )` — - src/actions.js:145
- Purpose / why used: Hydrates the store with fetched records and/or throws away stale cache via invalidateCache . Used internally by every resolver to commit fetched data.
- When used: After any REST fetch that returns entity records.
- Who uses it: The resolvers in resolvers.js .
- Inputs: kind, name, records (single or array), query, invalidateCache boolean, edits, meta. Output: dispatch of RECEIVE_ITEMS (+ optional RECEIVE_QUERIED_ITEMS ).

### `__experimentalBatch( requests )` — - src/actions.js:924
- Purpose / why used: Coalesces multiple REST requests into a single multipart __experimental/batch call using createBatch + defaultProcessor , resolving each sub-request's promise in order.
- When used: By callers that need several independent REST calls at once (e.g. bulk operations).
- Who uses it: block editor bulk utilities; performance-sensitive code paths.
- Inputs: array of { path, method, data } . Output: array of response promises.

### `getEntityRecord( kind, name, key, query )` — - src/resolvers.js:61
- Purpose / why used: Fetches a single entity record from the REST API and commits it via receiveEntityRecords . Also sets up the sync manager for sync-capable entities, requests per-record item permissions, and reads the Allow header for capability hints.
- When used: Automatically whenever getEntityRecord is selected and the record is not cached.
- Who uses it: Fired by @wordpress/data resolution; used by the editor for every single-entity read.
- Inputs: kind, name, key (id), query. Output: dispatches receiveEntityRecords and finishes resolution. shouldInvalidate clears the cache on template save/delete for root/site .

### `getEntityRecords( kind, name, query )` — - src/resolvers.js:356
- Purpose / why used: Fetches a collection of entity records (with pagination support), commits them via receiveEntityRecords (+ receiveQueriedItems ), stores the collection pagination meta, and requests collection-level permissions. In collaboration mode it also begins loading a sync collection.
- When used: Automatically when getEntityRecords is selected for an unresolved query.
- Who uses it: Fired by @wordpress/data resolution; used by lists, the pattern library, etc.
- Inputs: kind, name, query. Output: dispatches receiveEntityRecords . shouldInvalidate clears cache when RECEIVE_ITEMS / REMOVE_ITEMS carry the flag.

### `getCurrentUser() / getCurrentTheme() / getEmbedPreview() / canUser() / getAutosaves() / getRevisions() / getBlockPatterns() / getEntitiesConfig() / getThemeSupports() ...` — - src/resolvers.js:44-1311 (selected)
- Purpose / why used: The remaining resolvers each fetch one specialized payload and hydrate its cache slice: the current user ( /wp/v2/users/me ), the active theme + global-styles variants, embed preview data, permissions (OPTIONS on a resource), autosaves, revisions (with shouldInvalidate on save-finish), block patterns, and the dynamic entity configs ( getEntitiesConfig ).
- When used: Automatically when their matching selector is read and not cached.
- Who uses it: Fired by @wordpress/data resolution on first read.
- Inputs/Output: REST fetch + receive dispatch.

### `entity( entityConfig ) - Higher-Order reducer` — - src/reducer.js:178
- Purpose / why used: Builds the per-entity reducer, composed via withMultiEntityRecordEdits (hooks UNDO/REDO to replay edits as EDIT_ENTITY_RECORD ) + ifMatchingAction + replaceAction , wrapping a combineReducers({ queriedData, edits, saving, deleting, revisions }) .
- When used: Once per entity config, installed into the dynamic entities tree.
- Who uses it: The entities reducer builds these dynamically.
- Inputs: the entity config. Output: a dedicated reducer function.

### `entities( state, action ) - dynamic reducer tree` — - src/reducer.js:375
- Purpose / why used: Maintains the nested kind -> name -> entity(config) reducer tree and rebuilds it whenever the set of entity configs changes (new post types/taxonomies/site).
- When used: On ADD_ENTITIES and REMOVE_ITEMS -related config changes.
- Who uses it: Redux (root reducer); read by all entity selectors.
- Inputs: state, action. Output: updated entities state.

### `undoManager / syncUndoManagerState / userPermissions / embedPreviews / autosaves / blockPatterns ...` — - src/reducer.js:452-560 (selected)
- Purpose / why used: The remaining top-level slices: a singleton undoManager ( createUndoManager() , never mutated), syncUndoManagerState (Yjs/CRDT undo state), the userPermissions keyed cache, embedPreviews , autosaves , blockPatterns / blockPatternCategories / userPatternCategories , currentUser , currentTheme , navigationFallbackId , and defaultTemplates .
- When used: Each responds to its own action family (RECEIVE_*, *_RESOLVED, etc.).
- Who uses it: Redux (root reducer).
- Inputs: state, action. Output: updated slice.

### `getQueryParts( query )` — - src/queried-data/get-query-parts.js
- Purpose / why used: Normalizes a raw query object into a stable cache key and pagination parts: { stableKey, page, perPage, offset, include, fields, context } . Used by the reducer (query index keys) and selectors.
- When used: On every RECEIVE_QUERIED_ITEMS and every getQueriedItems call.
- Who uses it: queried-data/reducer.js , queried-data/selectors.js .
- Inputs: query object. Output: normalized parts object.

### `getMergedItemIds( itemIds, nextItemIds, page, perPage )` — - src/queried-data/reducer.js:30
- Purpose / why used: Merges two page-wise arrays of ids so paginated results accumulate across fetches without duplicates, using the current page + perPage to splice the new page into the right position.
- When used: Inside the queries reducer on RECEIVE_QUERIED_ITEMS .
- Who uses it: The queries reducer.
- Inputs: existing ids, new ids, page, perPage. Output: merged id array.

### `getQueriedItems( state, query, options )` — - src/queried-data/selectors.js:116
- Purpose / why used: Resolves the ids stored under a query's stable key into actual record objects, returning them as a filtered/mapped array. Caches by state reference + deep query equality via a WeakMap.
- When used: Inside getEntityRecords .
- Who uses it: getEntityRecords (selectors.ts).
- Inputs: the queriedData slice, query, options (key, isSticky). Output: array of records or undefined .

### `useEntityRecord( kind, name, id, options? )` — - src/hooks/use-entity-record.ts
- Purpose / why used: React hook returning a single entity record plus helpers: { record, editedRecord, hasEdits, save, edit, ... } .
- When used: In function components that read/edit one entity.
- Who uses it: Plugin/feature components.
- Inputs: kind, name, id, options. Output: the record + edit/save helpers.

### `useEntityRecords( kind, name, query?, options? )` — - src/hooks/use-entity-records.ts
- Purpose / why used: React hook returning a set of entity records for a query: { records, totalItems, totalPages, isResolving, ... } .
- When used: In function components that list entities.
- Who uses it: Plugin/feature components.
- Inputs: kind, name, query, options. Output: the records + pagination info.

### `useEntityProp( kind, name, id, prop )` — - src/hooks/use-entity-prop.js
- Purpose / why used: Returns [ value, setValue, fullValue ] for a single property of an entity, wiring a controlled input to getEditedEntityRecord + editEntityRecord .
- When used: For simple single-field editing forms.
- Who uses it: Editor field components.
- Inputs: kind, name, id, prop path. Output: the edit tuple.


---

# packages/components - full file & function map

**Owns:**
- The Context System : ContextSystemProvider , contextConnect() , useContextSystem() , and the polymorphic as rendering ( PolymorphicElement ).
- The full component inventory organized into one folder per component (each with index.tsx , types.ts , style.scss , README.md , and often hook.ts / component.tsx ).
- Design utilities in utils/ : colors, fonts, breakpoints, spacing, RTL, motion, math.
- The Higher-Order Components in higher-order/ : navigate-regions, withConstrainedTabbing, withFallbackStyles, withFilters, withFocusOutside, withFocusReturn, withNotices, withSpokenMessages.
- The SlotFill system ( slot-fill/ ): Slot , Fill , createSlotFill , SlotFillProvider .
- The newer Ariakit-based composed components (private): Menu , Tabs , Composite , Badge , and the validated form controls.

## File map / file details

- **src/index.ts** — Purpose: The single public barrel. Re-exports every component (with legacy and current names), the SVG primitives, the slot-fill system, the higher-order components, and the privateApis lock. This is the only public entry for @wordpress/components .
- **src/private-apis.ts · lock-unlock.js** — Purpose: Exposes the locked private surface via lock(privateApis, {...}) : ContentEditableControl , Menu , Tabs , Badge , ComponentsContext , useDrag , and the Validated*Control family. Reachable only through unlock() from other Gutenberg packages.
- **src/style.scss** — Purpose: Global stylesheet, pulls in theme-variables.scss (the --wp-components-* CSS custom properties) plus any package-wide base styles.
- **src/context/context-system-provider.tsx** — Purpose: ContextSystemProvider - a memoized React provider that deep-merges a value prop bag (namespaced component defaults) into the parent ComponentsContext . Exposes useComponentsContext() .
- **src/context/context-connect.ts** — Purpose: contextConnect(Component, namespace) (+ a -WithoutRef variant). Registers a component under a namespace by wrapping it in a ref-forwarding named wrapper and attaching a static CONNECT_STATIC_NAMESPACE symbol, plus displayName and a CSS selector.
- **src/context/use-context-system.js** — Purpose: useContextSystem(props, namespace) - the hook components call to merge context-injected defaults with direct props (direct props win), apply _overrides last, and compute the className.
- **src/context/get-styled-class-name-from-key.ts** — Purpose: Memoized getStyledClassNameFromKey(namespace) ; converts e.g. 'Button' to the CSS class 'components-button' .
- **src/context/wordpress-component.ts** — Purpose: TypeScript types for the polymorphic components: WordPressComponentProps , WordPressComponent , and the conditional as prop typing.
- **src/utils/config-values.js · colors-values.js · colors.js · font-values.js · font.js · space.ts** — Purpose: Design tokens: the CONFIG object (borders, radius, elevation, fonts), COLORS theme palette, the font('default.fontSize') resolver, the space() spacing scale, and color helpers like rgba() / getOptimalTextShade() .
- **src/utils/breakpoint.js · breakpoint-values.js · use-responsive-value.ts · rtl.js** — Purpose: Responsive + bidirectional layout: media-query breakpoint() helper and the rtl() helper that returns both LTR and RTL style objects.
- **src/utils/polymorphic-element.tsx** — Purpose: PolymorphicElement - the runtime engine behind the as prop. Filters incoming props through curated allowlists of valid HTML/SVG attributes so nothing invalid leaks to the DOM.
- **src/utils/style-mixins.js · theme-variables.scss · unit-values.ts · box-sizing.ts · dropdown-motion.ts** — Purpose: Shared style mixins/constants: boxSizingReset , baseLabelTypography , DROPDOWN_MOTION timing, CSS custom-property theming, and unit conversions.
- **src/utils/element-rect.ts · get-node-text.ts · get-valid-children.ts · strings.ts · math.js · values.js** — Purpose: DOM + data helpers: debounced getBoundingClientRect , extractTextContent , valid-children filtering, and string/math/value utilities.
- **src/utils/hooks/** — Purpose: Shared hooks: useCx (Emotion class merging), useControlledState / useControlledValue (controlled/uncontrolled pattern), useOnValueUpdate , useUpdateEffect .
- **src/slot-fill/index.tsx · slot.jsx · fill.jsx · use-slot.js** — Purpose: Slot (a named "location"), Fill (content registered into a slot), SlotFillProvider (context), and createSlotFill(name) (returns a { Slot, Fill } pair). This is the composition primitive that plugin Plugin* areas and toolbar registries are built on.

## Function notes

### `public barrel (index.ts)` — - src/index.ts:1-241
- Purpose / why used: Re-exports every public component, the SVG primitives, the slot-fill system, the higher-order components, and the privateApis . Because packages are bundled as wp.components.* globals and npm exports, this barrel is the single contract consumers import from.
- When used: On importing @wordpress/components .
- Who uses it: Every other editor package and third-party plugins.
- Inputs: none. Output: the full public namespace.

### `privateApis (private-apis.ts)` — - src/private-apis.ts:1-26
- Purpose / why used: Locks the internal-only components behind lock() so they can be used by other core packages but not by npm consumers until promoted to @wordpress/ui .
- When used: Imported by block-editor , editor , etc. via unlock( privateApis ) .
- Who uses it: Core editor packages.
- Inputs/Output: none - a locked object of component references.

### `ContextSystemProvider ({ value, children })` — - src/context/context-system-provider.tsx
- Purpose / why used: Provides a bag of namespaced default props to every ContextSystem -aware component in its subtree. This is how a parent theme/screen can style every Button , Menu , or Text at once without touching each usage.
- When used: At the root of editor screens / component trees to set default variants and sizes.
- Who uses it: Editor layout roots; the useContextSystem consumers.
- Inputs: value (e.g. { Button: { size: 'small' }, Text: { variant: 'muted' } } ), children. Output: a ComponentsContext.Provider .

### `contextConnect( Component, namespace, options? )` — - src/context/context-connect.ts
- Purpose / why used: Registers a component into the Context System under a namespace string. Instead of wrapping in an HOC, it creates a ref-forwarding named wrapper and attaches a static namespace marker.
- When used: At module load, to register every modern component (e.g. contextConnect(Text, 'Text') ).
- Who uses it: Component authors in this package.
- Inputs: component, namespace. Output: the registered component (with displayName , selector, and namespace static attached).

### `useContextSystem( props, namespace, shouldMergeComponents? )` — - src/context/use-context-system.js
- Purpose / why used: The core hook every modern component calls first. It merges the injected context defaults for namespace with the direct props (direct wins), applies _overrides last, computes the combined className via useCx , and returns the final props to spread onto the element.
- When used: Inside component hook.ts functions on every render.
- Who uses it: All ContextSystem -connected components (text, heading, view, spacer, ...).
- Inputs: props, namespace. Output: merged/derived props (including computed className).

### `PolymorphicElement ({ as = 'div', ...props })` — - src/utils/polymorphic-element.tsx
- Purpose / why used: Renders any element/component given via the as prop while filtering props through allowlists of valid HTML/SVG attributes, so invalid props never reach the DOM.
- When used: As the internal render target of the modern polymorphic components.
- Who uses it: View, Text, Heading, and other polymorphic components.
- Inputs: as (string or component), props. Output: a React element.

### `Modal / Popover / Dropdown` — - src/modal, src/popover, src/dropdown
- Purpose / why used: The three primary floating/overlay primitives: Modal renders a full-screen dialog (focus-trap via withConstrainedTabbing , scroll lock, overlay, close button, onRequestClose ); Popover positions a floating panel anchored to a trigger using @floating-ui /legacy placement, with arrow and flip logic; Dropdown composes Popover + focus handling + withFocusOutside into a toggleable popup.
- When used: Dialogs, menus, settings popovers, dropdowns throughout the editor.
- Who uses it: Editor UI; DropdownMenu , ToolbarDropdownMenu , Tooltip , color pickers built on Popover.
- Inputs: modal: title , onRequestClose , children; popover: anchor / position , children; dropdown: renderContent , renderToggle . Output: rendered overlay + portal.

### `DropdownMenu / Tooltip / Snackbar / Notice / Guide` — - src/dropdown-menu, src/tooltip, src/snackbar, src/notice, src/guide
- Purpose / why used: Higher-level overlay helpers: DropdownMenu (button + menu popover with menu items/controls), Tooltip (hover/focus popover text), Snackbar / SnackbarList (transient toasts + managed list), Notice (dismissible banner), Guide (multi-step onboarding modal with GuidePage children).
- When used: Context menus, help tooltips, save/error toasts, notices, welcome guides.
- Who uses it: Editor chrome; the @wordpress/notices store renders via SnackbarList / NoticeList .
- Inputs/Output: per-component props to rendered popup/toast/banner/modal.

### `Button / BaseControl / InputControl / SelectControl / FormTokenField / ...` — - src/button, src/base-control, src/input-control, src/select-control, src/form-token-field
- Purpose / why used: The core form-control family. Button is the workspace's most-used component (variants: primary/secondary/tertiary/link/is-destructive, sizes, icon-only, aria-pressed). BaseControl wraps a control with a label , help , and id binding. InputControl and the legacy TextControl / TextareaControl render text inputs. SelectControl / CustomSelectControl render native/custom selects. FormTokenField renders tokenized tag-style input. The Validated*Control family (private) adds built-in validation to these.
- When used: Everywhere a form control is needed.
- Who uses it: The whole editor, block supports controls, settings panels.
- Inputs: per-control props (value, onChange, label, disabled, ...). Output: the rendered DOM control, wired to onChange .

### `ColorPalette / ColorPicker / GradientPicker / FontSizePicker / RangeControl / ToggleControl / DateTimePicker` — - src/color-palette, src/color-picker, src/date-time, src/font-size-picker, ...
- Purpose / why used: Specialized pickers used by the block supports (color, gradient, font-size), the formatting toolbar, and settings panels. ColorPalette renders swatch options; ColorPicker provides a full HSV picker; GradientPicker lists gradient swatches + custom; FontSizePicker offers preset + custom sizes; RangeControl a slider (and supercedes NumberControl); ToggleControl an on/off switch; DateTimePicker date + time.
- When used: Block inspector color/font/focal settings; date publish scheduling.
- Who uses it: block-editor inspector controls, editor settings.
- Inputs/Output: value in + onChange(value) out.

### `layout primitive family (Card, Flex, Grid, HStack, VStack, ZStack, Spacer, Surface, Elevation, Text, Heading, View, Truncate, VisuallyHidden)` — - src/card, src/flex, src/grid, src/h-stack, src/v-stack, src/z-stack, src/spacer, ...
- Purpose / why used: The modern, polymorphic layout primitives, built on the Context System ( hook.ts + component.tsx + styles.ts with Emotion). Flex (and the HStack / VStack / ZStack shorthands) lay out children along an axis with gap / justify / wrap ; Grid does CSS grid; Card is the surface container with named parts; Spacer inserts flexible space; Surface / Elevation provide background/shadow; Text / Heading render typography with variants; View is the generic polymorphic box; Truncate ellipsizes; VisuallyHidden hides visually but keeps accessible.
- When used: Building new editor UI; these replace raw div s and SCSS with composition + tokens.
- Who uses it: Modern components and new editor screens; the @wordpress/ui migration target.
- Inputs: layout props (as, gap, align, justify, ...). Output: rendered polymorphic element with generated classes.

### `Panel / ItemGroup / Toolbar / ToolsPanel / TreeGrid / Placeholder` — - src/panel, src/item-group, src/toolbar, src/tools-panel, src/tree-grid, src/placeholder
- Purpose / why used: Structural containers: Panel (+ PanelBody collapsible, PanelHeader, PanelRow) is the classic settings block; ItemGroup groups list items; Toolbar (+ ToolbarButton, ToolbarGroup, ToolbarItem, ToolbarDropdownMenu) is the block formatting toolbar; ToolsPanel hosts toggleable control "tools" in the block inspector; TreeGrid renders a keyboard-navigable grid with RTL support; Placeholder shows empty-state blocks.
- When used: Inspector panels, block toolbar, settings grids, empty states.
- Who uses it: block-editor inspector + toolbar; edit-post settings.
- Inputs/Output: structural props + children.

### `Slot / Fill / SlotFillProvider / createSlotFill / useSlot / useSlotFills` — - src/slot-fill/index.tsx
- Purpose / why used: The composition mechanism that lets sibling/subtree components inject content into a named "slot" elsewhere in the tree, via React context + useSlots . This is how plugin areas, toolbar items, and notice lists are filled from anywhere.
- When used: Plugin sidebars, block settings menu fills, toolbar fills, notices.
- Who uses it: @wordpress/plugins , editor packages, and the slot-based registries.
- Inputs: Slot: name , fillProps , bubblesVirtually . Fill: name , children. Output: a slot location and its registered fills.

### `Icon / Dashicon / Spinner / ExternalLink / ClipboardButton / KeyboardShortcuts / Draggable / DropZone / Autocomplete / Animate` — - src/icon, src/dashicon, src/spinner, src/external-link, ...
- Purpose / why used: The remaining utility components: Icon renders an SVG icon (from @wordpress/icons or a spec); Dashicon renders a MDash icon font glyph; Spinner a loader; ExternalLink a safe external link with icon; ClipboardButton copies to clipboard on click; KeyboardShortcuts binds key handlers; Draggable HTML5 drag-and-drop wrapper; DropZone a drop target; Autocomplete provides suggestions in a text field; Animate adds show-in animation classes.
- When used: Toolbar icons, file/media drops, shortcuts, loading states.
- Who uses it: Editor chrome and block UI.
- Inputs/Output: component-specific props to rendered output.

### `Menu (compound) - Menu.Item / Menu.RadioItem / Menu.CheckboxItem / Menu.Group / Menu.GroupLabel / Menu.Separator / Menu.ItemLabel / Menu.ItemHelpText / Menu.Popover / Menu.TriggerButton / Menu.SubmenuTriggerItem` — - src/menu/index.tsx
- Purpose / why used: The next-generation dropdown menu built on Ariakit . It creates an Ariakit useMenuStore , provides it via Menu.Context , and assembles sub-components with Object.assign() . Supports proper ARIA roles, sub-menus, radio/checkbox items, labels, help text, and animated popovers. This supersedes the legacy flat MenuItem / MenuGroup / DropdownMenu .
- When used: New editor menus (block toolbar menus, options menus).
- Who uses it: block-editor/edit-site via unlock(privateApis) .
- Inputs: menu items as compound children. Output: an accessible menu with store-backed sub-parts.

### `Tabs (compound) - Tabs.Tab / Tabs.TabList / Tabs.TabPanel / Tabs.Context` — - src/tabs/index.tsx
- Purpose / why used: The next-generation tab component on Ariakit useTabStore , decomposing the legacy monolithic TabPanel into Tab , TabList , and TabPanel compound parts sharing a store via Tabs.Context .
- When used: New tabbed inspector/settings UI.
- Who uses it: Core packages via private API.
- Inputs: tabs as compound children. Output: accessible tab list + panels.

### `Composite - Composite.Group / Composite.GroupLabel / Composite.Item / Composite.Row / Composite.Hover / Composite.Typeahead` — - src/composite/index.tsx
- Purpose / why used: A generic single-tab-stop roving-tabindex widget backed by Ariakit useCompositeStore , replacing the hand-rolled navigable-container . Supports groups, rows, hover activation, and typeahead.
- When used: New keyboard-navigable collections.
- Who uses it: Core packages via private API.
- Inputs: items as compound children. Output: a roving-tabindex composite.

### `withNavigateRegions` — - src/higher-order/navigate-regions
- Purpose / why used: Wraps a component so the user can jump between UI regions using keyboard shortcuts (Ctrl+Shift+`/~, then keys 1-9); tracks regions and currentRegion .
- When used: Editor root to navigate between content/sidebar.
- Who uses it: edit-post / edit-site root layout.
- Inputs: wrapped component. Output: keyboard-navigable wrapper.

### `withConstrainedTabbing / withFocusReturn / withFocusOutside / withFallbackStyles / withFilters / withNotices / withSpokenMessages` — - src/higher-order/*
- Purpose / why used: The remaining focus/a11y/integration HOCs: withConstrainedTabbing traps Tab inside a focusable container (modal); withFocusReturn saves and restores the previously-focused element on mount/unmount; withFocusOutside calls handleFocusOutside when focus leaves the subtree (dropdowns, modals close); withFallbackStyles samples computed styles of a hidden node and passes them as props; withFilters exposes a component to @wordpress/hooks filters (plugin extension point); withNotices supplies noticeOperations + noticeUI ; withSpokenMessages injects speak / debouncedSpeak .
- When used: Modal (tab trap + focus return), dropdown (focus outside), plugin-extensible components (filters), form sections showing notices.
- Who uses it: Modal, Popover, Dropdown, editor panels, plugin areas.
- Inputs: wrapped component. Output: enhanced component.


---

# packages/block-editor - full file & function map

**Owns:**
- The store ( core/block-editor ): the block tree ( blocks slice), selection, editing modes, typing/dragging flags, settings, preferences, and template validity.
- The Block Supports system ( hooks/ ): every editor.BlockEdit / editor.BlockListBlock filter that injects per-support UI and markup.
- The block editing canvas : BlockList , BlockListBlock , BlockTools , BlockToolbar , WritingFlow , BlockPopover , and the iframe canvas.
- The inspector / settings / toolbar surfaces : inspector controls, block settings menu, block controls, tools panel.
- The inserter and list view subsystems.
- Rich text block editing ( RichText ), movement/drag-and-drop , link/media handling, and the block layout engine.

## File map / file details

- **src/index.js** — Purpose: The package entry point. Re-exports the style-support helpers ( __experimentalGet*ClassesAndStyles , use*Props ), everything from ./components , ./elements , and ./utils , the store + SETTINGS_DEFAULTS , and privateApis . Also imports ./hooks to register the block supports.
- **src/private-apis.js · lock-unlock.ts** — Purpose: The locked private surface: ExperimentalBlockEditorProvider , PrivateListView , PrivateQuickInserter , LayoutStyle , BlockManager , DimensionsTool , InspectorControlsLastItem , useZoomOut , global-styles engine private APIs, etc. Reachable only via unlock() .
- **src/elements/index.js** — Purpose: __experimentalGetElementClassName(element) - returns the theme element class for a given element (e.g. 'wp-element-button' , 'wp-element-caption' ). Used by the styles system to scope element styles.
- **src/utils/index.ts · src/autocompleters/block.jsx** — Purpose: transformStyles (CSS for the editor iframe) and getPxFromCssUnit ; and the "/" slash-command blocks autocompleter.
- **src/store/index.js · constants.js · defaults.js** — Purpose: storeConfig = { reducer, actions, selectors, persist: ['preferences'] } ; STORE_NAME = 'core/block-editor' ; SETTINGS_DEFAULTS (full editor settings) and PREFERENCES_DEFAULTS ( { insertUsage: {} } , the only persisted slice).
- **src/store/private-keys.js** — Purpose: Symbol keys (e.g. globalStylesDataKey , mediaEditKey , deviceTypeKey , isIsolatedEditorKey ) other packages use with unlock() to reach private APIs.
- **src/hooks/index.js** — Purpose: The single registration point: calls createBlockEditFilter([ ... ]) and createBlockListBlockFilter([ ... ]) over a list of support modules, hooking them into editor.BlockEdit and editor.BlockListBlock .
- **src/hooks/style.jsx** — Purpose: The core of the styles system. Merges global styles, compiles CSS via @wordpress/style-engine , renders the Typography/Dimensions/Border/Background inspector panels, and applies per-state (hover/pseudo) style handling through BlockStyleStateProvider .

## Function notes

### `store (create + register)` — - src/store/index.js:15-32
- Purpose / why used: Assembles the core/block-editor store config and registers it, plus the private actions/selectors.
- When used: On module load (importing @wordpress/block-editor ).
- Who uses it: All block-editor components and higher editor layers.
- Inputs: reducer + actions + selectors. Output: the registered store (exported as store ).

### `SETTINGS_DEFAULTS` — - src/store/defaults.js:35
- Purpose / why used: The default editor settings object: alignWide , supportsLayout , imageEditing / imageSizes , maxWidth , allowedBlockTypes , hasFixedToolbar , distractionFree , focusMode , styles , keepCaretInsideBlock , bodyPlaceholder , titlePlaceholder , canLockBlocks , codeEditingEnabled , plus responsive/zoom defaults.
- When used: As the initial settings slice; read by the whole system via getSettings / useSettings .
- Who uses it: The reducer, all canvas/toolbar/inspector components.
- Inputs/Output: none - a constant object.

### `blocks (the composite block-tree slice)` — - src/store/reducer.js:766
- Purpose / why used: The heart of the store. Models the block tree as flat maps keyed by client id: byClientId (block objects), attributes , order (parent -> child-id array), parents (child -> parent), controlledInnerBlocks , and blockEditingModes . Handles every tree-mutation action (insert/remove/move/replace/update/merge).
- When used: On every block-tree action.
- Who uses it: The root reducer; consumed by every block-tree selector.
- Inputs: state, action. Output: new block-tree state.

### `The other state slices (selection, flags, UI)` — - src/store/reducer.js (multiple lines)
- Purpose / why used: The remaining top-level slices track the editing UI state: isBlockInterfaceHidden (1239), isTyping (1259), isDragging (1280), draggedBlocks (1300), blockVisibility (1320), selection (start/end, 1405), isMultiSelecting (1505), isSelectionEnabled (1525), initialPosition (1615), blocksMode (1636), insertionCue (1661), template + templateLock (1698), settings (1718), preferences (1757), blockListSettings (1810), lastBlockAttributesChange (1884), highlightedBlock (1916), hasBlockSpotlight (1945), expandedBlock (1984), lastBlockInserted (2005), styleOverrides (2076), lastFocus (2114), zoomLevel (2131), insertionPoint (2150), listViewContentPanelOpen (2240), requestedInspectorTab (2265), selectedBlockStyleState (2287), styleStateViewport (2397), isResponsiveEditing (2414).
- When used: Each responds to its own action family (± dragging, ± typing, set selection, etc.).
- Who uses it: Redux (root reducer); read by the corresponding public/private selectors.
- Inputs: state, action. Output: updated slice.

### `withDerivedBlockEditingModes (HOC reducer)` — - src/store/reducer.js:2873
- Purpose / why used: A higher-order reducer that computes derived block editing modes (for synced patterns and zoomed-out mode) and stores them in state.derivedBlockEditingModes , so the base slices can stay simple while consumption sees an effective editing mode.
- When used: Wraps the whole combined reducer.
- Who uses it: The root reducer; getBlockEditingMode selectors.
- Inputs: inner reducer. Output: a reducer that enriches state.

### `getBlock(s) / getBlockByClientId / getBlockName / getBlockAttributes / getBlockOrder / getBlockIndex / getBlockParents / getBlockRootClientId / getBlockCount / getPreviousBlockClientId / getNextBlockClientId / getAdjacentBlockClientId` — - src/store/selectors.js:101-430+
- Purpose / why used: The block-tree read API - navigate and inspect the in-memory block tree: getBlock(clientId) returns a full block object; getBlocks() / getBlockRootClientId / getBlockParents navigate ancestry; getBlockOrder lists a parent's children; getBlockIndex finds a sibling position; getPrevious/Next/AdjacentBlockClientId walk siblings.
- When used: Everywhere the tree structure is needed (rendering, insertion-point math, mover).
- Who uses it: BlockList, BlockListBlock, BlockMover, the list view, keyboard shortcuts, etc.
- Inputs: state (+ clientId/parent). Output: block data / id / order.

### `Selection selectors: getSelectedBlock / getSelectedBlockClientId / getMultiSelectedBlockClientIds / isBlockSelected / hasSelectedInnerBlock / getSelectionStart / getSelectionEnd / isMultiSelecting / isBlockMultiSelected / getFirstMultiSelectedBlockClientId / getLastMultiSelectedBlockClientId` — - src/store/selectors.js:442-1270
- Purpose / why used: The selection model read API. Returns the currently selected client id / block, the multi-selection range, whether a given block is selected/multi-selected, the caret's selection start/end (RichText), and the multi-select flag.
- When used: To drive focus, chrome visibility, toolbar state, and range operations.
- Who uses it: BlockListBlock, BlockToolbar, block inspector, keyboard shortcuts, RichText.
- Inputs: state (+ clientId). Output: selection info.

### `Capability selectors: canInsertBlockType / canInsertBlocks / canRemoveBlock(s) / canMoveBlock(s) / canEditBlock / canLockBlockType` — - src/store/selectors.js:1956-2442
- Purpose / why used: Decide what operations are allowed for a block in its current context (parent, mode, template lock, editing mode, block "lock" attribute). These gate the UI (hide/disable move, remove, insert).
- When used: In the mover buttons, block settings menu, appenders, inserter.
- Who uses it: BlockMover, BlockSettingsMenu, appenders, the inserter.
- Inputs: state, block name(s), rootClientId, options. Output: boolean.

### `Inserter/pattern selectors: getInserterItems / getAllowedBlocks / getBlockTransformItems / getPatternsByBlockTypes / getDirectInsertBlock` — - src/store/selectors.js:2443-3007
- Purpose / why used: Compute the items (blocks, patterns, variations) available to insert at an insertion point, respecting allowed-block filtering, search, and transforms.
- When used: By the inserter subsystem and pattern library.
- Who uses it: Inserter menu, quick inserter, block pattern panels.
- Inputs: state, rootClientId, insertionIndex, filterValue, ... Output: item/pattern arrays.

### `Settings/status selectors: getSettings / getBlockListSettings / isTyping / isDraggingBlocks / getBlockMode / isCaretWithinFormattedText / isValidTemplate / getBlockEditingMode / isBlockVisible / didAutomaticChange / getBlockInsertionPoint ...` — - src/store/selectors.js:1458-3528 (selected)
- Purpose / why used: The remaining readers for settings, per-block list settings (~3.2), interactions (typing, dragging, caret), template validity, the effective block editing mode, block visibility, automatic changes (undo-ignored), and the computed insertion point.
- When used: Throughout the canvas and chrome.
- Who uses it: block-tools, writing-flow, toolbar, inspector.
- Inputs/Output: state (+ block id) to the cached value.

### `resetBlocks / receiveBlocks / replaceBlock(s) / insertBlock(s) / insertBlocks / removeBlock(s) / replaceInnerBlocks / moveBlock(s)ToPosition / moveBlocksUp/Down / duplicateBlocks / insertBeforeBlock / insertAfterBlock / insertDefaultBlock / updateBlockAttributes / updateBlock / mergeBlocks` — - src/store/actions.js:45-2019
- Purpose / why used: The block-tree write API. Every structural change (create, insert, remove, reorder, copy, merge, split, attribute update) flows through these thunks, which dispatch the matching reducer action and set up undo/redo levels and automatic-change flags.
- When used: Whenever the block tree changes (user edits, paste, transforms, programmatic updates).
- Who uses it: BlockListBlock, inserter, block mover, copy/handler, transforms, RichText split/merge, the editor layer.
- Inputs: blocks / clientIds / attributes / positions. Output: dispatched reducer actions.

### `Selection actions: selectBlock / selectNextBlock / selectPreviousBlock / multiSelect / clearSelectedBlock / startMultiSelect / stopMultiSelect / selectionChange / toggleSelection / hoverBlock / flashBlock / toggleBlockHighlight` — - src/store/actions.js:208-373,1683-2019
- Purpose / why used: Change the current selection (single, multi, caret), clear it, toggle selection enabled, or momentarily highlight/flash a block (for "scroll to block" animations).
- When used: Click/select gestures, keyboard navigation, "reveal"/list-view selection, programmatic focus.
- Who uses it: BlockListBlock, writing-flow, block-tools, list view, movers.
- Inputs: clientId / start/end / toSelect. Output: dispatched actions.

### `startTyping / stopTyping / startDraggingBlocks / stopDraggingBlocks / enterFormattedText / exitFormattedText / updateSettings / updateBlockListSettings / toggleBlockMode / setTemplateValidity / showInsertionPoint / hideInsertionPoint / synchronizeTemplate` — - src/store/actions.js:1585-2019 (selected)
- Purpose / why used: Interaction + settings actions: type/drag state flags; focused RichText (formatted text) state; update global settings or per-block list settings; toggle block "HTML mode"; mark a template valid/invalid and sync it; show/hide the insertion-point cue.
- When used: On typing/dragging/focus, settings updates, template validation.
- Who uses it: WritingFlow, block-tools, the editor layer, settings panels.
- Inputs/Output: flags/settings to dispatched actions.

### `Private actions (registered via registerPrivateActions)` — - src/store/private-actions.js:33-614
- Purpose / why used: Internal actions reachable only via unlock() : __experimentalUpdateSettings , hideBlockInterface / showBlockInterface , privateRemoveBlocks , ensureDefaultBlock , setBlockRemovalRules , setStyleOverride / deleteStyleOverride , setInsertionPoint , editContentOnlySection / stopEditingContentOnlySection , setZoomLevel / resetZoomLevel , toggleBlockSpotlight , list-view panel open/close/toggle, showViewportModal , requestInspectorTab , setSelectedBlockStyleState . Used by the extensible site editor, canvas, zoom modes, and the block style/hover states.
- When used: By core editor packages via unlock( store ) .
- Who uses it: edit-post/edit-site/v2 route packages, block-canvas, block-tools.
- Inputs/Output: state mutations for protected features.

### `createBlockEditFilter( supports ) / createBlockListBlockFilter( supports )` — - src/hooks/index.js (via hooks/utils)
- Purpose / why used: The registration engine. For each support module it adds two WordPress filters: editor.BlockEdit (to inject inspector UI and edit-time behavior) and editor.BlockListBlock (to inject the wrapper's className / data attributes). This is how a block type's declared supports become real UI and styling with no per-block code.
- When used: Once, at module load, over the list of all support modules.
- Who uses it: hooks/index.js ; the filter callbacks run whenever a block's BlockEdit or BlockListBlock renders.
- Inputs: the array of support modules { edit, attributeKeys, hasSupport } . Output: registered filters.

### `style.jsx - the styles core (BlockSupports)` — - src/hooks/style.jsx
- Purpose / why used: The most important support. Reads the block's style attribute and the global styles, compiles CSS via @wordpress/style-engine ( getCSSRules / compileCSS / getCSSValueFromRawStyle ), and provides the inspector panels (Typography, Dimensions, Border, Background). Handles state-based (hover/pseudo) styles via BlockStyleStateProvider and scopes selectors with buildScopedBlockSelector .
- When used: On every block render (to compute className/style) and in the inspector.
- Who uses it: All blocks with a style support; the global-styles controls.
- Inputs: block attributes + settings/global styles. Output: compiled CSS + inspector UI.

### `use*Props helpers: useColorProps / useBorderProps / useSpacingProps / useTypographyProps / useDimensionsProps / useShadowProps, and get*ClassesAndStyles` — - src/hooks/use-*-props.js
- Purpose / why used: Pure helpers (no hook/HOC) that convert a block's style attributes into CSS class names and inline style objects for the block wrapper. Publicly exported as __experimentalGet*ClassesAndStyles / __experimentalUse*Props for block authors.
- When used: By block authors and the supports system to compute colors/borders/spacing markup.
- Who uses it: Block authors via the public API; the supports hooks internally.
- Inputs: attributes (color, border, spacing, typography) + settings. Output: { className, style } to spread onto the wrapper.

### `supports.js - the support testers` — - src/hooks/supports.js
- Purpose / why used: Named predicates to query a block type's supports declaration without stringly-typed lookups: hasAlignSupport , getAlignSupport , hasBorderSupport , hasColorSupport , hasLinkColorSupport , hasGradientSupport , hasBackgroundColorSupport , hasTextAlignSupport , hasFontFamilySupport , hasFontSizeSupport , hasLayoutSupport , getLayoutSupport , hasStyleSupport , hasCustomClassNameSupport , etc.
- When used: To decide whether to render specific inspector controls.
- Who uses it: Global-styles panels and various inspector components.
- Inputs: block type name. Output: boolean / support value.

### `BlockList (root) / BlockListBlock / useBlockProps` — - src/components/block-list/index.jsx, block.jsx, use-block-props
- Purpose / why used: The rendering surface. BlockList (Root) is the recursive container that renders the in-between inserter, sets up layout context, block-selection clearing, and the zoom-out separator. BlockListBlock ( block.jsx ) renders a single block: it applies all BlockListBlock support filters, computes the wrapper via useBlockProps , and wires selection/dragging/toolbars. useBlockProps is the hook block authors call to get the block wrapper's className/style props.
- When used: The whole block editing area; every block render.
- Who uses it: The editor canvas; InnerBlocks nests more BlockList .
- Inputs: blocks in client-id order. Output: the rendered, interactive block tree.

### `useWritingFlow` — - src/components/writing-flow/index.jsx
- Purpose / why used: The keyboard + selection model that makes the canvas behave like a continuous writing surface: it handles arrow keys moving between blocks, Enter to insert, Backspace/Delete merges, Tab (with withConstrainedTabbing ), click-through selection, drag, and the multi-select gesture. Composed of ~16 sub-hooks (selection, dragging, click-through, keyboard navigation, tab key, etc.).
- When used: Wraps the whole BlockList in the provider.
- Who uses it: The Provider /experimental-provider wraps children.
- Inputs: registered event handlers. Output: keyboard/selection behavior across blocks.

### `ExperimentalBlockEditorProvider` — - src/components/provider/index.jsx
- Purpose / why used: The boot component. Wires every cross-cutting provider the editor needs: SlotFillProvider , withRegistryProvider , useBlockSync , BlockRefsProvider , SelectionContext , MediaUploadProvider , KeyboardShortcuts , BlockKeyboardShortcuts , and media-upload settings (including client-side HEIC processing). Also handles the CSS/style synchronization from the settings into the canvas.
- When used: The root of any block editing context (post editor, widgets, site editor, FSE).
- Who uses it: @wordpress/editor , edit-post , the widgets screen, site editor.
- Inputs: value (settings), children , useSubRegistry . Output: a fully-wired provider tree.

### `Inserter subsystem` — - src/components/inserter/
- Purpose / why used: The block insertion UI. Inserter opens a panel (and there is a quick inserter variant, private). Sub-parts: Menu (the panel), Library , tabs for block types / block patterns / media, SearchItems , SearchResults , PreviewPanel , Tips , and hooks: use-insertion-point ( useInsertionPoint computes where to insert), use-block-types-state , use-patterns-state , use-patterns-paging . Supports dragging to insert via inserter-draggable-blocks .
- When used: When the user opens the "+" inserter or `/` command.
- Who uses it: block-tools appender, block-toolbar, the command palette.
- Inputs: rootClientId + insertionIndex. Output: the inserter panel & insertion logic.

### `RichText / PlainText / EditableText` — - src/components/rich-text, plain-text, editable-text
- Purpose / why used: RichText is the WYSIWYG text-editing component for block content, wrapping @wordpress/rich-text 's editing infrastructure with block-editor integration: it renders a contentEditable, tracks selection, supports inline formats/toolbar, autocompletes (via useBlockEditorAutocompleteProps ), and syncs the value back through onChange . PlainText is the raw single-line input used by blocks like a code/cover; EditableText is the internal editable wrapper. Event-listeners (paste, inputs, insert) live under rich-text/ .
- When used: Every block's editable text content.
- Who uses it: The core/richtext, core/paragraph, core/heading, etc. blocks.
- Inputs: value , onChange , multiline , allowedFormats . Output: the contentEditable + toolbar + formatting.

### `InnerBlocks` — - src/components/inner-blocks/index.jsx
- Purpose / why used: The component container blocks use to render their nested blocks. It renders a BlockList for the block's inner client-ids, registers an appender, and binds to a specific rootClientId . Provides the useInnerBlocksProps hook for flexible nesting.
- When used: In any block with inner content (group, columns, cover, navigation).
- Who uses it: All container blocks.
- Inputs: clientId, renderAppender, template. Output: the nested block list.

### `InspectorControls / InspectorAdvancedControls / BlockControls / BlockToolbar / BlockSettingsMenu` — - src/components/inspector-controls, inspector-controls-tabs, block-controls, block-toolbar, block-settings-menu
- Purpose / why used: The settings/toolbar surface built on Slot/Fill: InspectorControls (Fill) registers content tabs in the inspector sidebar; InspectorAdvancedControls the advanced tab; tabs are content/styles/settings via inspector-controls-tabs . BlockControls (and BlockFormatControls ) fill the toolbar. BlockToolbar renders the floating/contextual toolbar with mover, switcher, and more. BlockSettingsMenu is the "..." dropdown (convert to HTML, rename, group, lock, move, duplicate, remove).
- When used: Selected block chrome and the inspector sidebar.
- Who uses it: block-tools, block-edit, the editor's inspector.
- Inputs/Output: Slot/Fill children to the rendered toolbar/inspector.

### `List View` — - src/components/list-view/
- Purpose / why used: The outline sidebar showing the full block hierarchy with drag/drop reorder, expand/collapse, renaming, locking, and selection sync. Exported as PrivateListView (private) because it needs the isolated-editor context.
- When used: The "List view" panel in the editor.
- Who uses it: edit-post/edit-site list-view panel; block-editor's own useListViewController .
- Inputs/Output: block tree in, interactive outline out.

### `layoutTypes (flow / constrained / flex / grid) + getLayoutType / getLayoutTypes` — - src/layouts/index.js, flow.js, constrained.jsx, flex.jsx, grid.jsx, definitions.js
- Purpose / why used: Defines the layouts that determine how inner blocks flow. flow is the default ( getLayoutStyle() computes block-gap and alignment CSS); constrained caps content/widht (dividing content vs. wide sizes, using GLOBAL_CONTENT_SIZE / GLOBAL_WIDE_SIZE ); flex and grid lay out children along an axis/grid. getLayoutType(name) / getLayoutTypes() resolve a layout by name. Parity is kept with block-supports/layout.php .
- When used: When rendering a block list to compute container styles and default inner-block alignment.
- Who uses it: block-list/layout ( LayoutProvider , LayoutStyle ), the child-layout control, gap settings.
- Inputs: layout attribute/name. Output: layout CSS / supported info.


---

# packages/edit-post — full file & function map

**Owns:**
- The boot sequence ( initializeEditor ) that mounts the whole editor.
- The root Layout component that assembles every other screen part.
- The meta box subsystem (the only Redux state this store really keeps: metaBoxes ).
- The entity navigation stack (post ↔ template / pattern navigation).
- The browser URL sync and the classic-revision redirect.
- Legacy SlotFill re-exports in deprecated.jsx for backward compatibility.

## File map / file details

- **src/index.jsx** — Purpose: The package entry point. Exports initializeEditor() (boots the editor), reinitializeEditor() (deprecated noop), the legacy BackButton aliases, the store , and everything from deprecated.jsx . This is what wp-admin's edit.php script calls.
- **src/deprecated.jsx** — Purpose: Re-exports the 8 classic Plugin* SlotFill components from @wordpress/editor , but logs a deprecation notice and renders null inside the site editor. Kept so old plugins that extend the post editor via wp.editPost.PluginX still work.
- **src/lock-unlock.js** — Purpose: Opts into the private-APIs system and exports the lock / unlock functions used to reach the private ( unstable ) APIs of @wordpress/editor , @wordpress/api-fetch , @wordpress/components , etc. from within core modules.
- **src/classic.scss · src/style.scss** — Purpose: Stylesheets. classic.scss styles the classic editor chrome; style.scss holds the package's main styles used across the editor screen.
- **src/store/index.js** — Purpose: Defines and registers the Redux store named core/edit-post via createReduxStore + register , wiring together its reducer, actions, and selectors.
- **src/store/constants.js** — Purpose: Holds STORE_NAME = 'core/edit-post' plus two CSS selector strings used to find the admin-bar "View"/"Preview" links ( #wp-admin-bar-view a , #wp-admin-bar-preview a ).
- **src/store/reducer.js** — Purpose: The reducer. It keeps a single metaBoxes slice composed of isSaving , locations , and initialized . This is the only state core/edit-post truly owns.
- **src/store/actions.js** — Purpose: All dispatchable actions. Many (sidebar, modal, panel, publish-sidebar, preview device, inserter, etc.) are now deprecated wrappers that delegate to another store; the genuinely local ones are the meta box actions ( requestMetaBoxUpdates , initializeMetaBoxes , etc.) and the fullscreen / feature toggles.
- **src/store/selectors.js** — Purpose: All selectors (read api). Most are deprecated wrappers delegating to other stores; the local ones concern meta boxes ( getActiveMetaBoxLocations , hasMetaBoxes , etc.) and a few feature/panel helpers.
- **src/components/layout/index.jsx ★ ROOT** — Purpose: The Layout component — the container that renders the entire editor: registers commands, sets up entity navigation, initializes meta boxes, computes editor styles, and renders Editor plus all overlays ( MetaBoxesMain , BrowserURL , WelcomeGuide , MoreMenu , BackButton , PluginArea , notifications).
- **src/components/back-button/index.jsx · fullscreen-mode-close.jsx** — Purpose: The "Back" button shown when fullscreen is active. Renders the classic BackButton fill containing FullscreenModeClose , a Button that links back to the post list ( edit.php ).
- **src/components/browser-url/index.js + use-classic-revision-redirect.js** — Purpose: Keeps the browser's address bar in sync with the currently open post/revision (via history.replaceState , debounced). A companion hook redirects to the classic revision.php screen when visual revisions are disabled.
- **src/components/editor-initialization/index.js + listener-hooks.js** — Purpose: A data-only component (renders null ) that runs post-load listeners — most importantly updating the admin-bar "View post / Preview" link's href and visibility whenever the permalink changes.
- **src/components/init-pattern-modal/index.jsx** — Purpose: When a user creates a new wp_block (a pattern), shows a modal to name it and choose whether it is synced or unsynced, then calls editPost with the title and pattern sync meta.
- **src/components/keyboard-shortcuts/index.js** — Purpose: Registers the core/edit-post/toggle-fullscreen shortcut (Secondary+F) and binds it to dispatch toggleFullscreenMode .
- **src/components/meta-boxes/ ★ subsystem** — Purpose: The meta box subsystem: renders each meta box area, toggles box visibility on the page, initializes WordPress's classic postboxes script, and saves/restores meta boxes around post save. Files: index.jsx , meta-box-visibility.js , use-meta-box-initialization.js , meta-boxes-area/index.jsx , meta-boxes-area/style.scss .
- **src/components/more-menu/index.jsx · manage-patterns-menu-item.jsx · welcome-guide-menu-item.jsx** — Purpose: The editor's "Options" ( … ) drop-down menu. Adds the Fullscreen mode toggle, a "Manage patterns" link, a "Welcome Guide" re-open item, and opens the Preferences modal.
- **src/components/preferences-modal/index.jsx · enable-custom-fields.jsx · enable-panel.jsx · meta-boxes-section.jsx** — Purpose: The Settings (Preferences) modal. Adds the "Use theme styles" toggle, the "Advanced" meta box section (custom fields toggle + per-meta-box panel toggles).
- **src/components/welcome-guide/index.jsx · default.jsx · template.jsx · image.jsx** — Purpose: The first-run "Welcome" guide. Picks the default (post) guide or the template-editor guide depending on post type, and renders a multi-page Guide .
- **src/hooks/use-navigate-to-entity-record.js** — Purpose: A useReducer -backed navigation stack that records the "entity history" as the user moves between editing a post and its template/pattern, preserving block selection and rendering mode. Powers the Back button and the template-editing flow.
- **src/commands/use-commands.js** — Purpose: Registers the core/toggle-fullscreen-mode command in the command palette (Ctrl+K), toggling the fullscreen preference and showing an info snackbar with an Undo action.
- **src/utils/meta-boxes.js** — Purpose: A single DOM helper — getMetaBoxContainer(location) — that returns the current meta box area DOM node for a location (or the original #metaboxes container).

## Function notes

### `initializeEditor( id, postType, postId, settings, initialEdits )` — · src/index.jsx:43
- Purpose / why used: Bootstraps the entire post editor from scratch and mounts it into the page. This is THE entry point to the classic editor.
- When used: Once, when wp-admin loads edit.php for a post.
- Who uses it: The WordPress wp-admin JS bootstrap (the script enqueued on edit.php ).
- Inputs: id (DOM element id), postType (e.g. 'post' ), postId (post ID), settings (editor settings object incl. template , styles ), initialEdits (programmatic edits to apply on load, treated as non-user-initiated).
- Output: A React root ( createRoot ), and — after preload resolves — it renders the <Layout> component into the DOM target. This rendered UI is what the user sees.

### `preloadResolutions( postType, postId )` — · src/index.jsx:165
- Purpose / why used: Drives the data resolvers to completion against the preload cache before React mounts, so the first paint has finished metadata (no stale "loading" flash). Runs in 3 phases because some requests depend on the state derived from earlier ones.
- When used: Inside initializeEditor , before root.render .
- Who uses it: Only initializeEditor .
- Inputs: postType , postId .
- Output: A Promise<void> . Its resolution args are used to set up the editor via setupEditor (through .finally() in the caller).

### `reinitializeEditor()` — · src/index.jsx:306
- Purpose / why used: Used to reinitialize the editor after an error. It is now a deprecated noop that only logs a deprecation warning.
- When used: Was called after a fatal error to remount; now never does anything.
- Who uses it: Historically internal; now only surface for backward-compat callers.
- Inputs: None. Output: undefined .

### `deprecateSlot( name )` — · src/deprecated.jsx:21
- Purpose / why used: Shared helper that logs a deprecation notice telling plugins wp.editPost.X → wp.editor.X .
- When used: Every time one of the deprecated slot components renders.
- Who uses it: All the components in this file.
- Inputs: name (slot name). Output: none (side-effect deprecation log).

### `PluginBlockSettingsMenuItem / PluginDocumentSettingPanel / PluginMoreMenuItem / PluginPrePublishPanel / PluginPostPublishPanel / PluginPostStatusInfo / PluginSidebar / PluginSidebarMoreMenuItem / __experimentalPluginPostExcerpt` — · src/deprecated.jsx:32–130
- Purpose / why used: Backward-compatible re-exports of @wordpress/editor 's SlotFill components under the old wp.editPost names. They render null inside the site editor and log a deprecation otherwise. Keeps legacy plugins working while steering them to wp.editor.* .
- When used: Whenever a plugin still renders one of these SlotFills.
- Who uses it: Third-party plugins (slot-fill consumers).
- Inputs: props (forwarded to the corresponding EditorPlugin* component). Output: the editor plugin component (or null in the site editor, or PluginPostExcerpt for the excerpt one).

### `lock-unlock module` — · src/lock-unlock.js:3
- Purpose / why used: Opts into the private-APIs gatekeeper and exposes lock / unlock so core modules can reach unstable APIs of other packages.
- When used: Imported by many files in this package to reach @wordpress/editor , @wordpress/api-fetch , @wordpress/components , etc.
- Who uses it: index.jsx, store/actions.js, store/selectors.js, components/layout, etc.
- Inputs: none. Output: { lock, unlock } .

### `store (register)` — · src/store/index.js:14
- Purpose / why used: Creates and registers the core/edit-post Redux store.
- When used: On module load (importing the package or index.jsx exports the store).
- Who uses it: All components and hooks in the package via @wordpress/data .
- Inputs: reducer + actions + selectors. Output: the registered store descriptor (exported as store ).

### `STORE_NAME / VIEW_AS_LINK_SELECTOR / VIEW_AS_PREVIEW_LINK_SELECTOR` — · src/store/constants.js:6,13,20
- Purpose / why used: Defines the store identifier and the admin-bar CSS selectors used by the editor-initialization listener.
- When used: Throughout the package (whose store name), and in listener-hooks.js (the selectors).
- Who uses it: store/index.js, editor-initialization/listener-hooks.js.
- Inputs: none. Output: constant strings.

### `isSavingMetaBoxes( state, action )` — · src/store/reducer.js:13
- Purpose / why used: Reducer for the "meta boxes are currently being saved" boolean. true means a save request is in flight.
- When used: On REQUEST_META_BOX_UPDATES , META_BOX_UPDATES_SUCCESS , META_BOX_UPDATES_FAILURE .
- Who uses it: Redux dispatch from requestMetaBoxUpdates ; read by isSavingMetaBoxes selector (→ MetaBoxesArea spinner).
- Inputs: previous state, action. Output: new boolean state.

### `mergeMetaboxes( metaboxes, newMetaboxes )` — · src/store/reducer.js:25
- Purpose / why used: Merges incoming meta boxes into the existing list, updating by id if it already exists, else appending.
- When used: Inside metaBoxLocations on SET_META_BOXES_PER_LOCATIONS .
- Who uses it: The metaBoxLocations reducer.
- Inputs: current list, new list. Output: merged list.

### `metaBoxLocations( state, action )` — · src/store/reducer.js:51
- Purpose / why used: Reducer for the locations map (location → meta box list).
- When used: On SET_META_BOXES_PER_LOCATIONS .
- Who uses it: Redux (part of metaBoxes ). Read by meta box selectors.
- Inputs: state, action. Output: updated locations object.

### `metaBoxesInitialized( state, action )` — · src/store/reducer.js:78
- Purpose / why used: Reducer for the "meta boxes initialized" flag.
- When used: On META_BOXES_INITIALIZED (from initializeMetaBoxes ).
- Who uses it: Redux; read by areMetaBoxesInitialized selector.
- Inputs: state, action. Output: boolean.

### `requestMetaBoxUpdates()` — · src/store/actions.js:276
- Purpose / why used: Saves all classic meta boxes via an AJAX POST to window._wpMetaBoxUrl . This is the only async save path for classic PHP meta boxes.
- When used: On post save (hooked to the editor.savePost action), except for autosaves, when meta boxes exist.
- Who uses it: Dispatched from initializeMetaBoxes 's addAction('editor.savePost', ...) handler. Its result is read by the isSavingMetaBoxes selector → MetaBoxesArea spinner.
- Inputs: none directly (reads DOM .metabox-base-form , meta box containers, and edits from core-data for backward-compat fields like comment/ping/sticky/author).
- Output: dispatches META_BOX_UPDATES_SUCCESS or META_BOX_UPDATES_FAILURE , which sets isSaving false and is observed via isSavingMetaBoxes .

### `initializeMetaBoxes()` — · src/store/actions.js:454
- Purpose / why used: Initializes WordPress's classic postboxes script (so meta boxes are expandable/draggable) and wires up the meta box save hook ( editor.savePost ). Only runs once, and only once the editor is ready.
- When used: From useMetaBoxInitialization when the editor is ready and meta boxes are active.
- Who uses it: The useMetaBoxInitialization hook (via useDispatch(editPostStore) ).
- Inputs: none. Output: dispatches META_BOXES_INITIALIZED and attaches the editor.savePost hook that calls requestMetaBoxUpdates .

### `metaBoxUpdatesSuccess() / metaBoxUpdatesFailure()` — · src/store/actions.js:352,363
- Purpose / why used: Minimal actions reporting the outcome of a meta box save.
- When used: After the AJAX save resolves / rejects, from requestMetaBoxUpdates .
- Who uses it: requestMetaBoxUpdates ; consumed by the isSavingMetaBoxes reducer.
- Inputs: none. Output: action objects { type: 'META_BOX_UPDATES_SUCCESS' } / { type: 'META_BOX_UPDATES_FAILURE' } .

### `setAvailableMetaBoxesPerLocation( metaBoxesPerLocation )` — · src/store/actions.js:266
- Purpose / why used: Stores which meta boxes are available in which location.
- When used: When the server reports the meta boxes per location.
- Who uses it: Called by the backend data-loading path; consumed by metaBoxLocations reducer and read by meta box selectors ( getActiveMetaBoxLocations , etc.).
- Inputs: metaBoxesPerLocation object. Output: { type: 'SET_META_BOXES_PER_LOCATIONS', metaBoxesPerLocation } .

### `toggleFeature( feature ) / toggleFullscreenMode()` — · src/store/actions.js:185,511
- Purpose / why used: toggleFeature toggles a core/edit-post preference flag; toggleFullscreenMode toggles the fullscreenMode preference, shows a snackbar notice (with an Undo action), and is the target of the fullscreen keyboard shortcut and command.
- When used: toggleFeature — anywhere toggling a feature; toggleFullscreenMode — keyboard shortcut (Secondary+F), the More menu item, and the command palette.
- Who uses it: toggleFeature → welcome guide finish, etc.; toggleFullscreenMode → keyboard-shortcuts/index.js and commands/use-commands.js. Output read by preferences selectors ( isFeatureActive ).
- Inputs: feature (toggleFeature) or none (toggleFullscreenMode). Output: dispatches to @wordpress/preferences and @wordpress/notices .

### `openGeneralSidebar / closeGeneralSidebar / togglePinnedPluginItem / showBlockTypes / hideBlockTypes` — · src/store/actions.js:22,33,214,244,255
- Purpose / why used: Not-yet-deprecated thin wrappers that delegate to other stores: sidebar open/close → core/interface ; pinned plugin toggle → core/interface ; show/hide block types → core/editor private actions.
- When used: When those UI operations happen in the classic editor.
- Who uses it: Editor UI components; output read by the corresponding interface/editor selectors.
- Inputs/Outputs: name/pluginName/blockNames in, delegation to another store out.

### `getActiveMetaBoxLocations( state )` — · src/store/selectors.js:350
- Purpose / why used: Returns the list of meta box locations that actually have active (non-empty) meta boxes.
- When used: Whenever the UI needs to know which meta box areas to render/save.
- Who uses it: requestMetaBoxUpdates (to gather form data), hasMetaBoxes , and other meta box selectors. Its output drives which areas render and are included in the save payload.
- Inputs: state. Output: string[] of locations (e.g. ['normal','side'] ).

### `isMetaBoxLocationActive( state, location )` — · src/store/selectors.js:389
- Purpose / why used: Returns true if a given location has at least one meta box.
- When used: Inside getActiveMetaBoxLocations and isMetaBoxLocationVisible .
- Who uses it: The selectors above; isMetaBoxLocationVisible combines it with enabled panels.
- Inputs: state, location. Output: boolean.

### `getMetaBoxesPerLocation( state, location ) / getAllMetaBoxes( state )` — · src/store/selectors.js:402,413
- Purpose / why used: Reads the meta box list for one location, or flattens all locations into one array.
- When used: getMetaBoxesPerLocation → MetaBoxes component; getAllMetaBoxes → the Preferences modal's meta box section and RTC-compatibility checks.
- Who uses it: components/meta-boxes, components/preferences-modal, hooks/use-meta-box-initialization.
- Inputs: state (+ location). Output: array of meta box objects [{ id, title, ... }] .

### `isMetaBoxLocationVisible( state, location )` — · src/store/selectors.js:367
- Purpose / why used: Returns true when a location is both active and has at least one of its meta boxes panel-enabled (i.e. visibly enabled in the UI).
- When used: MetaBoxesMain in layout/index.jsx to decide whether to show the meta box pane.
- Who uses it: Layout 's MetaBoxesMain .
- Inputs: state, location. Output: boolean.

### `hasMetaBoxes( state ) / isSavingMetaBoxes( state ) / areMetaBoxesInitialized( state )` — · src/store/selectors.js:427,438,539
- Purpose / why used: Convenience readers: does the post use meta boxes (any active location); are meta boxes currently saving; are they initialized.
- When used: hasMetaBoxes → save hook + layout; isSavingMetaBoxes → MetaBoxesArea spinner; areMetaBoxesInitialized → init checks.
- Who uses it: initializeMetaBoxes (save hook), MetaBoxesArea, use-meta-box-initialization, Layout.
- Inputs: state. Output: booleans.

### `isFeatureActive( state, feature )` — · src/store/selectors.js:322
- Purpose / why used: Returns whether a core/edit-post preference/feature is active.
- When used: Layout (fullscreen, theme styles, welcome guide), More menu, WelcomeGuide.
- Who uses it: Layout 's useEditorStyles and main select; WelcomeGuide .
- Inputs: state, feature. Output: boolean.

### `getEditorMode( state ) / isEditorSidebarOpened( state ) / isPluginSidebarOpened( state ) / getActiveGeneralSidebarName( state ) / getHiddenBlockTypes( state ) / isPluginItemPinned( state, pluginName ) / getPreference( ... ) / getPreferences( state ) / getEditedPostTemplate( state )` — · src/store/selectors.js:21–198,205–341,548
- Purpose / why used: Thin read wrappers around other stores (preferences/interface/editor): editing mode, sidebar open state, active sidebar name, hidden block types, pinned items, preferences (converted panel format), and the template of the edited post.
- When used: By various UI components in the package and by plugins.
- Who uses it: Layout and assorted components; getPreferences / getPreference are deprecated but kept for plugins.
- Inputs/Outputs: state (+ params) in; delegated value out (boolean/string/object).

### `Layout( { postId, postType, settings, initialEdits } )` — · src/components/layout/index.jsx:349
- Purpose / why used: The root component of the whole editor screen. It wires commands, entity navigation, meta box initialization, editor styles, and renders the full tree: SlotFillProvider → ThemeProvider → ErrorBoundary → <Editor> plus all overlays.
- When used: Once, rendered by initializeEditor .
- Who uses it: initializeEditor (root.render). It consumes settings from the server and renders the user-facing editor.
- Inputs: postId , postType , settings , initialEdits .
- Output: the rendered editor UI (React element tree). Inputs' edits flow into Editor ; the computed editorSettings (incl. resolved styles ) are passed to Editor .

### `useEditorStyles( settings )` — · src/components/layout/index.jsx:71
- Purpose / why used: Computes the list of editor styles to inject, merging default editor styles with preset styles and (when supported and not overridden) theme styles, plus fallback layout styles.
- When used: Each render of Layout .
- Who uses it: Layout ; its output array feeds editorSettings.styles → Editor , which applies the styles to the editing canvas.
- Inputs: settings . Output: string[] of resolved style objects.

### `MetaBoxesMain()` — · src/components/layout/index.jsx:120
- Purpose / why used: The resizable normal + advanced meta box pane at the bottom of the editor. Handles open/close toggle, drag-to-resize (via useDrag ), height persistence in preferences, and ARIA separator semantics.
- When used: Rendered by Layout as extraContent when meta boxes are visible and not distraction-free.
- Who uses it: Layout . It renders <MetaBoxes location="normal" /> and <MetaBoxes location="advanced" /> .
- Inputs/outputs: none directly; reads/writes preferences; renders the meta box areas.

### `MetaBoxes( { location } )` — · components/meta-boxes/index.jsx:6
- Purpose / why used: Renders all meta boxes for a given location. For each box it renders a MetaBoxVisibility (to show/hide it) and a shared MetaBoxesArea .
- When used: For normal / advanced (in MetaBoxesMain ) and side (in Layout's extraSidebarPanels ).
- Who uses it: MetaBoxesMain and Layout .
- Inputs: location . Output: rendered meta box area in the DOM.

### `MetaBoxVisibility( { id } )` — · components/meta-boxes/meta-box-visibility.js:5
- Purpose / why used: Adds/removes the is-hidden class on a meta box's DOM element based on whether its panel is enabled.
- When used: When a box's enabled state or existence changes.
- Who uses it: MetaBoxes ; the DOM element it toggles is the meta box node the user sees.
- Inputs: id . Output: DOM side effect (class toggle).

### `useMetaBoxInitialization( enabled )` — · components/meta-boxes/use-meta-box-initialization.js:13
- Purpose / why used: Orchestrates meta box setup when the editor is ready: calls initializeMetaBoxes , disables real-time collaboration when incompatible (non-RTC) meta boxes exist, and disables visual revisions when any meta box is active (since classic meta box values can't be restored by the visual revision restore).
- When used: From Layout on mount, with enabled = hasActiveMetaboxes && hasResolvedMode .
- Who uses it: Layout .
- Inputs: enabled (boolean). Output: side effects — dispatches initializeMetaBoxes , setCollaborationSupported(false) , updateEditorSettings({ disableVisualRevisions: true }) .

### `MetaBoxesArea( { location } )` — · components/meta-boxes/meta-boxes-area/index.jsx:36
- Purpose / why used: Renders the meta box area container and moves the classic meta box form node into it (preserving iframe/subtree state via moveBefore ). Shows a spinner while meta boxes save, and grants the pane native browser undo/redo.
- When used: Rendered by MetaBoxes for each location. The move helper runs on mount and cleanup.
- Who uses it: MetaBoxes . The moved formRef node is the classic .metabox-location-<location> form from the server.
- Inputs: location . Output: rendered area + DOM relocation of the meta box form.

### `getPostEditURL( postId, revisionId )` — · components/browser-url/index.js:22
- Purpose / why used: Builds the post.php?... edit URL for a post, optionally including a revision id.
- When used: Inside BrowserURL 's effect to write the URL.
- Who uses it: BrowserURL . Its output becomes the browser address via history.replaceState .
- Inputs: postId , optional revisionId . Output: URL string.

### `BrowserURL()` — · components/browser-url/index.js:30
- Purpose / why used: Keeps the address bar in sync with the open post/revision. Reads the initial revision from the URL (opening it if present), then updates the URL via history.replaceState , debounced (300 ms) to avoid overwhelming Safari's History API limits.
- When used: On mount and whenever post id / status / revision changes.
- Who uses it: Rendered by Layout . Output: the window.location.href (via replaceState).
- Inputs: none directly (reads editor store + window.location ). Output: browser URL side effect; returns null .

### `useClassicRevisionRedirect()` — · components/browser-url/use-classic-revision-redirect.js:13
- Purpose / why used: Redirects the editor URL to the classic revision.php screen when visual revisions are disabled and the URL carries a revision id — but only after verifying the revision belongs to the current post (via a resolveSelect), to avoid redirecting on a bad/expired URL.
- When used: Called inside BrowserURL .
- Who uses it: BrowserURL . Output: window.location.replace to revision.php .
- Inputs: none (reads editor settings + core-data). Output: navigation side effect.

### `BackButton( { initialPost } )` — · components/back-button/index.jsx:16
- Purpose / why used: Wraps the editor's back-button fill and, when it's the only item in the stack, renders a motion-animated FullscreenModeClose button (slides in when fullscreen).
- When used: In Layout when fullscreen is active and viewport is ≥ medium.
- Who uses it: Layout . Output: back button in the UI.
- Inputs: initialPost . Output: rendered button (or nothing).

### `FullscreenModeClose( { showTooltip, icon, href, initialPost } )` — · components/back-button/fullscreen-mode-close.jsx:9
- Purpose / why used: Renders the "Back" button that links to the post list ( edit.php ) for the post's type, using a chevron icon (RTL-aware).
- When used: Inside BackButton .
- Who uses it: BackButton . Output: an anchor Button navigating to the admin list.
- Inputs: showTooltip , icon , href , initialPost . Output: Button with href = edit.php?post_type=...<slug> .

### `EditorInitialization()` — · components/editor-initialization/index.js:9
- Purpose / why used: A data-only component that runs post-load listeners and renders nothing. Re-init happens when postId changes or on unmount.
- When used: Rendered inside <Editor> in Layout .
- Who uses it: Layout . Output: none (returns null ); triggers listeners.

### `useUpdatePostLinkListener()` — · components/editor-initialization/listener-hooks.js:14
- Purpose / why used: Watches the current permalink and updates the admin-bar "View post / Preview" link's href (and shows/hides it when the post isn't viewable).
- When used: On mount (capture node) and whenever permalink/visibility changes.
- Who uses it: EditorInitialization . Output: DOM updates to #wp-admin-bar-preview a / #wp-admin-bar-view a .

### `InitPatternModal()` — · components/init-pattern-modal/index.jsx:13
- Purpose / why used: For a new, clean wp_block (pattern) post, shows a modal to name the pattern and choose whether it is synced or unsynced, then saves via editPost .
- When used: Rendered by Layout only when currentPostType === 'wp_block' , and only when it's a clean new post.
- Who uses it: Layout . Output: form submit → editPost({ title, meta }) in @wordpress/editor .
- Inputs: none (reads isCleanNewPost ). Output: a Modal (or null ).

### `KeyboardShortcuts()` — · components/keyboard-shortcuts/index.js:10
- Purpose / why used: Registers the core/edit-post/toggle-fullscreen shortcut (Modifier+Shift+F or secondary+F) and binds it to dispatch toggleFullscreenMode .
- When used: Rendered by Layout ; registers on mount.
- Who uses it: Layout . Output: keyboard shortcut registration + handler.

### `MoreMenu()` — · components/more-menu/index.jsx:23
- Purpose / why used: The editor's "Options" ( … ) menu. Adds the Fullscreen mode toggle (large viewports), "Manage patterns" and "Welcome Guide" items, and opens the Preferences modal.
- When used: Rendered by Layout inside <Editor> .
- Who uses it: Layout . Output: menu group items.

### `ManagePatternsMenuItem()` — · components/more-menu/manage-patterns-menu-item.jsx:10
- Purpose / why used: A "Manage patterns" menu item linking to the patterns screen in the site editor, or to edit.php?post_type=wp_block if the user lacks the capability.
- When used: Inside MoreMenu .
- Who uses it: MoreMenu . Output: a menu item whose href is capability-dependent.

### `WelcomeGuideMenuItem()` — · components/more-menu/welcome-guide-menu-item.jsx:12
- Purpose / why used: A "Welcome Guide" menu item that toggles the welcome-guide preference (template variant when editing a template).
- When used: Inside MoreMenu .
- Who uses it: MoreMenu . Output: toggles the preference (re-shows the guide).

### `EditPostPreferencesModal()` — · components/preferences-modal/index.jsx:10
- Purpose / why used: Wraps the shared Preferences modal, adding the "Advanced" meta box section and the "Use theme styles" appearance toggle.
- When used: Opened from MoreMenu .
- Who uses it: MoreMenu . Output: the Settings modal UI.

### `MetaBoxesSection( sectionProps )` — · components/preferences-modal/meta-boxes-section.jsx:12
- Purpose / why used: The "Advanced" settings section. Shows the Custom Fields enable toggle (special-cased out of the general list) and a toggle per third-party meta box panel. Hides entirely if there is nothing to show.
- When used: Inside EditPostPreferencesModal .
- Who uses it: EditPostPreferencesModal . Output: preference toggle controls.

### `EnableCustomFieldsOption( { label } ) + CustomFieldsConfirmation( { willEnable } ) + submitCustomFieldsForm()` — · components/preferences-modal/enable-custom-fields.jsx:12,25,53
- Purpose / why used: A preference option for toggling custom fields. Because enabling/disabling custom fields requires a page reload, it shows a CustomFieldsConfirmation with a "Show & Reload Page" button that submits the #toggle-custom-fields-form (updating its _wp_http_referer first).
- When used: Inside MetaBoxesSection when custom fields are registered.
- Who uses it: MetaBoxesSection . Output: a confirmation and a form-submit button.

### `EnablePanelOption( props )` — · components/preferences-modal/enable-panel.jsx:8
- Purpose / why used: A generic preference toggle for enabling/disabling an editor panel (used for each third-party meta box), hiding itself if the panel was programmatically removed.
- When used: Inside MetaBoxesSection .
- Who uses it: MetaBoxesSection . Output: toggle that calls toggleEditorPanelEnabled .

### `WelcomeGuide( { postType } )` — · components/welcome-guide/index.jsx:6
- Purpose / why used: Chooses and renders the first-run welcome guide — the default post guide or the template guide — based on whether it is active in preferences and the post type.
- When used: Rendered by Layout on mount.
- Who uses it: Layout . Output: a WelcomeGuideDefault or WelcomeGuideTemplate .

### `WelcomeGuideDefault() / WelcomeGuideTemplate() / WelcomeGuideImage( { nonAnimatedSrc, animatedSrc } )` — · components/welcome-guide/{default,template,image}.jsx
- Purpose / why used: The default guide is a 4-page Guide teaching blocks; the template guide is a 1-page guide for the template editor. Each finishes by toggling its welcome feature off. WelcomeGuideImage renders a a <picture> that switches to a static image under reduced-motion.
- When used: From WelcomeGuide .
- Who uses it: WelcomeGuide . Output: Guide pages; finishing toggles the feature off.

### `useNavigateToEntityRecord( initialPostId, initialPostType, defaultRenderingMode )` — · hooks/use-navigate-to-entity-record.js:21
- Purpose / why used: Tracks "entity history" as the user navigates between editing a post and editing its template/pattern. Implements a stack (like browser history): pushing a new entity saves the current block selection and rendering mode; popping restores them. Powers the Back button and the post → template → back flow.
- When used: In Layout on mount.
- Who uses it: Layout . Output: { currentPost, onNavigateToEntityRecord, onNavigateToPreviousEntityRecord } — fed into Editor as handlers; the previous-handler drives the back button.
- Inputs: initialPostId , initialPostType , defaultRenderingMode . Output: the navigation API object above.

### `useCommands()` — · commands/use-commands.js:8
- Purpose / why used: Registers the core/toggle-fullscreen-mode command in the command palette. When run, it toggles the fullscreen preference, closes the palette, and shows a snackbar notification with an Undo action.
- When used: From Layout on mount (via useEditPostCommands ).
- Who uses it: Layout . Output: a registered command in @wordpress/commands .

### `getMetaBoxContainer( location )` — · utils/meta-boxes.js:10
- Purpose / why used: Returns the current meta box DOM node for a location, preferring the one inside the editor's meta box area, falling back to the original #metaboxes container.
- When used: In requestMetaBoxUpdates to gather each location's form data.
- Who uses it: requestMetaBoxUpdates (store/actions.js). Output: a DOM element used to build the save payload.
- Inputs: location . Output: DOM node (or null ).


---

# One-shot responsibility map

| Package | Primary responsibility |
|---|---|
| `@wordpress/data` | State-management infrastructure: registries, Redux stores, selectors, actions, resolvers, React bindings, persistence. |
| `@wordpress/blocks` | Block API: registration, creation, transforms, parsing, serialization, validation, raw/paste handling, `core/blocks`. |
| `@wordpress/rich-text` | Rich text value model and transformations, DOM/HTML conversion, format registration/store, RichText hooks/listeners. |
| `@wordpress/core-data` | WordPress entity data layer: entity configuration, REST-backed records, selectors/actions/resolvers, queried-data cache. |
| `@wordpress/components` | Shared reusable UI components: context, overlays, forms, layout, composition, composed components, HOCs. |
| `@wordpress/block-editor` | Generic interactive block-editing surface: block tree state, selection, supports, canvas, inserter, list view, inspector, toolbar, layouts, movement. |
| `@wordpress/edit-post` | Full wp-admin post-editor orchestration: bootstrap, root layout, meta boxes, entity navigation, browser URL synchronization, legacy integrations. |

## Debugging boundary

`state infrastructure → data`  
`block definition/object → blocks`  
`formatted text → rich-text`  
`WordPress entity/API data → core-data`  
`reusable UI → components`  
`interactive block editing → block-editor`  
`whole wp-admin editor screen → edit-post`
