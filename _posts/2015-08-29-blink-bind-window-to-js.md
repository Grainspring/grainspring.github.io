---
layout: post
title:  "Blink/WebKit Bind C++ Window To Javascript"
date:   2015-08-29 16:06:05
categories: Blink Javascript
excerpt: Blink How To Work Series。
---

* content
{:toc}

## V8WindowShell
V8WindowShell represents all the per-global object state for a Frame that persist between navigations.

###Class Define
	55 class V8WindowShell {
	56 public:
	57     static PassOwnPtr<V8WindowShell> create(Frame*, PassRefPtr<DOMWrapperWorld>, v8::Isolate*);
	59     v8::Local<v8::Context> context() const { return m_context.newLocal(m_isolate); }
	61     // Update document object of the frame.
	62     void updateDocument();
	80     DOMWrapperWorld* world() { return m_world.get(); }
	82 private:
	83     V8WindowShell(Frame*, PassRefPtr<DOMWrapperWorld>, v8::Isolate*);
	85     void disposeContext();
	95
	96     void createContext();
	97     bool installDOMWindow();
	98
	99     static V8WindowShell* enteredIsolatedWorldContext();
	100
	101     Frame* m_frame;
	102     RefPtr<DOMWrapperWorld> m_world;
	103     v8::Isolate* m_isolate;
	104
	105     OwnPtr<V8PerContextData> m_perContextData;
	106
	107     ScopedPersistent<v8::Context> m_context; // pay attention here ScopedPersistent for m_context.
	108     ScopedPersistent<v8::Object> m_global; // also to m_global.
	109     ScopedPersistent<v8::Object> m_document; // also to m_document.
	110 };

###Construct Function
We should provide a frame and world and isolate to construct it.
	81 PassOwnPtr<V8WindowShell> V8WindowShell::create(Frame* frame, PassRefPtr<DOMWrapperWorld> world, v8::Isolate* isolate)
	82 {
	83     return adoptPtr(new V8WindowShell(frame, world, isolate));
	84 }
	85
	86 V8WindowShell::V8WindowShell(Frame* frame, PassRefPtr<DOMWrapperWorld> world, v8::Isolate* isolate)
	87     : m_frame(frame)
	88     , m_world(world)
	89     , m_isolate(isolate)
	90 {
	91 }

###initializeIfNeeded
	147 // Create a new environment and setup the global object.
	148 //
	149 // The global object corresponds to a DOMWindow instance. However, to
	150 // allow properties of the JS DOMWindow instance to be shadowed, we
	151 // use a shadow object as the global object and use the JS DOMWindow
	152 // instance as the prototype for that shadow object. The JS DOMWindow
	153 // instance is undetectable from JavaScript code because the __proto__
	154 // accessors skip that object.
	155 //
	156 // The shadow object and the DOMWindow instance are seen as one object
	157 // from JavaScript. The JavaScript object that corresponds to a
	158 // DOMWindow instance is the shadow object. When mapping a DOMWindow
	159 // instance to a V8 object, we return the shadow object.
	160 //
	161 // To implement split-window, see
	162 //   1) https://bugs.webkit.org/show_bug.cgi?id=17249
	163 //   2) https://wiki.mozilla.org/Gecko:SplitWindow
	164 //   3) https://bugzilla.mozilla.org/show_bug.cgi?id=296639
	165 // we need to split the shadow object further into two objects:
	166 // an outer window and an inner window. The inner window is the hidden
	167 // prototype of the outer window. The inner window is the default
	168 // global object of the context. A variable declared in the global
	169 // scope is a property of the inner window.
	170 //
	171 // The outer window sticks to a Frame, it is exposed to JavaScript
	172 // via window.window, window.self, window.parent, etc. The outer window
	173 // has a security token which is the domain. The outer window cannot
	174 // have its own properties. window.foo = 'x' is delegated to the
	175 // inner window.
	176 //
	177 // When a frame navigates to a new page, the inner window is cut off
	178 // the outer window, and the outer window identify is preserved for
	179 // the frame. However, a new inner window is created for the new page.
	180 // If there are JS code holds a closure to the old inner window,
	181 // it won't be able to reach the outer window via its global object.
	182 bool V8WindowShell::initializeIfNeeded()
	183 {
	184     if (!m_context.isEmpty())
	185         return true;
	186
	187     v8::HandleScope handleScope(m_isolate);
	188
	189     V8Initializer::initializeMainThreadIfNeeded(m_isolate); //1.do v8 initialize.
	190
	191     createContext(); //2.create v8::Context.
	192     if (m_context.isEmpty())
	193         return false;
	194
	195     v8::Handle<v8::Context> context = m_context.newLocal(m_isolate);
	196
	197     m_world->setIsolatedWorldField(context);
	199     bool isMainWorld = m_world->isMainWorld();
	200
	201     v8::Context::Scope contextScope(context);
	202
	203     if (m_global.isEmpty()) {
	204         m_global.set(m_isolate, context->Global()); //3.get/set global set scope persist.
	205         if (m_global.isEmpty()) {
	206             disposeContext();
	207             return false;
	208         }
	209     }
	210
	211     if (!isMainWorld) {
	212         V8WindowShell* mainWindow = m_frame->script()->existingWindowShell(mainThreadNormalWorld());
	213         if (mainWindow && !mainWindow->context().IsEmpty())
	214             setInjectedScriptContextDebugId(context, m_frame->script()->contextDebugId(mainWindow->context()));
	215     }
	216
	217     m_perContextData = V8PerContextData::create(context); //4.create V8PerContextData.
	218     if (!m_perContextData->init()) {
	219         disposeContext();
	220         return false;
	221     }
	222     m_perContextData->setActivityLogger(DOMWrapperWorld::activityLogger(m_world->worldId()));
	223     if (!installDOMWindow()) { //5.install dom window to context.
	224         disposeContext();
	225         return false;
	226     }
	227
	228     if (isMainWorld) {
	229         updateDocument(); //6.set docoument to context
	230         setSecurityToken();
	231         if (m_frame->document()) {
	232             ContentSecurityPolicy* csp = m_frame->document()->contentSecurityPolicy();
	233             context->AllowCodeGenerationFromStrings(csp->allowEval(0, ContentSecurityPolicy::SuppressReport));
	234             context->SetErrorMessageForCodeGenerationFromStrings(v8String(csp->evalDisabledErrorMessage(), m_isolate));
	235         }
	236     } else {
	237         // Using the default security token means that the canAccess is always
	238         // called, which is slow.
	239         // FIXME: Use tokens where possible. This will mean keeping track of all
	240         //        created contexts so that they can all be updated when the
	241         //        document domain
	242         //        changes.
	243         context->UseDefaultSecurityToken();
	244
	245         SecurityOrigin* origin = m_world->isolatedWorldSecurityOrigin();
	246         if (origin && InspectorInstrumentation::hasFrontends()) {
	247             ScriptState* scriptState = ScriptState::forContext(v8::Local<v8::Context>::New(context));
	248             InspectorInstrumentation::didCreateIsolatedContext(m_frame, scriptState, origin);
	249         }
	250     }
	251     m_frame->loader()->client()->didCreateScriptContext(context, m_world->extensionGroup(), m_world->worldId());
	252     return true;
	253 }

###createContext
	255 void V8WindowShell::createContext()
	256 {
	257     // The activeDocumentLoader pointer could be 0 during frame shutdown.
	258     // FIXME: Can we remove this check?
	259     if (!m_frame->loader()->activeDocumentLoader())
	260         return;
	261
	262     // Create a new environment using an empty template for the shadow
	263     // object. Reuse the global object if one has been created earlier.
	264     v8::Handle<v8::ObjectTemplate> globalTemplate = V8Window::GetShadowObjectTemplate(m_isolate, m_world->isMainWorld() ? MainWorld : IsolatedWorld);
	265     if (globalTemplate.IsEmpty())
	266         return;
	267
	268     double contextCreationStartInSeconds = currentTime();
	288     ..............................................................
	289     v8::HandleScope handleScope(m_isolate);
	290     m_context.set(m_isolate, v8::Context::New(m_isolate, &extensionConfiguration, globalTemplate, m_global.newLocal(m_isolate))); //pay attention to here, for how to v8::Context::New.
	291
	292     double contextCreationDurationInMilliseconds = (currentTime() - contextCreationStartInSeconds) * 1000;
	293     const char* histogramName = "WebCore.V8WindowShell.createContext.MainWorld";
	294     if (!m_world->isMainWorld())
	295         histogramName = "WebCore.V8WindowShell.createContext.IsolatedWorld";
	296     HistogramSupport::histogramCustomCounts(histogramName, contextCreationDurationInMilliseconds, 0, 10000, 50);
	297 }

###installDOMWindow
	299 bool V8WindowShell::installDOMWindow()
	300 {
	301     DOMWrapperWorld::setInitializingWindow(true);
	302     DOMWindow* window = m_frame->domWindow();
	303     v8::Local<v8::Object> windowWrapper = V8ObjectConstructor::newInstance(V8PerContextData::from(m_context.newLocal(m_isolate))->constructorForType(&V8Window::info));
	304     if (windowWrapper.IsEmpty()) // get real windowWrapper for DOMWindow.
	305         return false;
	306
	307     V8Window::installPerContextProperties(windowWrapper, window, m_isolate);
	308
	309     V8DOMWrapper::setNativeInfo(v8::Handle<v8::Object>::Cast(windowWrapper->GetPrototype()), &V8Window::info, window);
	310
	311     // Install the windowWrapper as the prototype of the innerGlobalObject.
	312     // The full structure of the global object is as follows:
	313     //
	314     // outerGlobalObject (Empty object, remains after navigation)
	315     //   -- has prototype --> innerGlobalObject (Holds global variables, changes during navigation)
	316     //   -- has prototype --> DOMWindow instance
	317     //   -- has prototype --> Window.prototype
	318     //   -- has prototype --> Object.prototype
	319     //
	320     // Note: Much of this prototype structure is hidden from web content. The
	321     //       outer, inner, and DOMWindow instance all appear to be the same
	322     //       JavaScript object.
	323     //
	324     v8::Handle<v8::Object> innerGlobalObject = toInnerGlobalObject(m_context.newLocal(m_isolate)); //v8::Handle<v8::Object>::Cast(context->Global()->GetPrototype());
	325     V8DOMWrapper::setNativeInfo(innerGlobalObject, &V8Window::info, window);
	326     innerGlobalObject->SetPrototype(windowWrapper);//set windowWrapper as context->Global()'s prototype.
	327     V8DOMWrapper::associateObjectWithWrapper<V8Window>(PassRefPtr<DOMWindow>(window), &V8Window::info, windowWrapper, m_isolate, WrapperConfiguration::Dependent);
	328     DOMWrapperWorld::setInitializingWindow(false);
	329     return true;
	330 }

###updateDocumentProperty
	338 void V8WindowShell::updateDocumentProperty()
	339 {
	340     if (!m_world->isMainWorld())
	341         return;
	342
	343     v8::HandleScope handleScope(m_isolate);
	344     v8::Handle<v8::Context> context = m_context.newLocal(m_isolate);
	345     v8::Context::Scope contextScope(context);
	346
	347     v8::Handle<v8::Value> documentWrapper = toV8(m_frame->document(), v8::Handle<v8::Object>(), context->GetIsolate());
	348     ASSERT(documentWrapper == m_document.newLocal(m_isolate) || m_document.isEmpty());
	349     if (m_document.isEmpty())
	350         updateDocumentWrapper(v8::Handle<v8::Object>::Cast(documentWrapper));
	351     checkDocumentWrapper(m_document.newLocal(m_isolate), m_frame->document());
	352
	353     // If instantiation of the document wrapper fails, clear the cache
	354     // and let the DOMWindow accessor handle access to the document.
	355     if (documentWrapper.IsEmpty()) {
	356         clearDocumentProperty();
	357         return;
	358     }
	359     ASSERT(documentWrapper->IsObject());
	360     context->Global()->ForceSet(v8::String::NewSymbol("document"), documentWrapper, static_cast<v8::PropertyAttribute>(v8::ReadOnly | v8::DontDelete));
	361
	362     // We also stash a reference to the document on the inner global object so that
	363     // DOMWindow objects we obtain from JavaScript references are guaranteed to have
	364     // live Document objects.
	365     toInnerGlobalObject(context)->SetHiddenValue(V8HiddenPropertyName::document(), documentWrapper);
	366 }

###clearForClose
	112 void V8WindowShell::clearForClose(bool destroyGlobal)
	113 {
	114     if (destroyGlobal)
	115         m_global.clear();
	116
	117     if (m_context.isEmpty())
	118         return;
	119
	120     m_document.clear();
	121     disposeContext();
	122 }
	
	93 void V8WindowShell::disposeContext()
	94 {
	95     m_perContextData.clear();
	96
	97     if (m_context.isEmpty())
	98         return;
	99
	100     v8::HandleScope handleScope(m_isolate);
	101     m_frame->loader()->client()->willReleaseScriptContext(m_context.newLocal(m_isolate), m_world->worldId());
	102
	103     m_context.clear();
	104
	105     // It's likely that disposing the context has created a lot of
	106     // garbage. Notify V8 about this so it'll have a chance of cleaning
	107     // it up when idle.
	108     bool isMainFrame = m_frame->page() && (m_frame->page()->mainFrame() == m_frame);
	109     V8GCForContextDispose::instance().notifyContextDisposed(isMainFrame);
	110 }

## V8PerContextData
###Class Define
	61 class V8PerContextData {
	62 public:
	63     static PassOwnPtr<V8PerContextData> create(v8::Handle<v8::Context> context)
	64     {
	65         return adoptPtr(new V8PerContextData(context));
	66     }
	68     ~V8PerContextData()
	69     {
	70         dispose();
	71     }
	73     bool init();
	75     static V8PerContextData* from(v8::Handle<v8::Context> context)
	76     {
	77         return static_cast<V8PerContextData*>(context->GetAlignedPointerFromEmbedderData(v8ContextPerContextDataIndex));
	78     }
	79 ....................................................
	89     v8::Local<v8::Function> constructorForType(WrapperTypeInfo* type)
	90     {
	91         UnsafePersistent<v8::Function> function = m_constructorMap.get(type);
	92         if (!function.isEmpty())
	93             return function.newLocal(v8::Isolate::GetCurrent());
	94         return constructorForTypeSlowCase(type);
	95     }
	96 ...................................................
	116 private:
	117     explicit V8PerContextData(v8::Handle<v8::Context> context)
	118         : m_activityLogger(0)
	119         , m_isolate(v8::Isolate::GetCurrent())
	120         , m_context(m_isolate, context) //m_context as v8::Persistent.
	121         , m_customElementBindings(adoptPtr(new CustomElementBindingMap()))
	122     {
	123     }
	124
	125     void dispose();
	134     ...................................................
	135     typedef WTF::HashMap<WrapperTypeInfo*, UnsafePersistent<v8::Function> > ConstructorMap;
	136     ConstructorMap m_constructorMap;
	137
	138     V8NPObjectMap m_v8NPObjectMap;
	139     // We cache a pointer to the V8DOMActivityLogger associated with the world
	140     // corresponding to this context. The ownership of the pointer is retained
	141     // by the DOMActivityLoggerMap in DOMWrapperWorld.
	142     V8DOMActivityLogger* m_activityLogger;
	143     v8::Isolate* m_isolate;
	144     v8::Persistent<v8::Context> m_context;
	145     ScopedPersistent<v8::Value> m_errorPrototype;
	146
	147     typedef WTF::HashMap<CustomElementDefinition*, OwnPtr<CustomElementBinding> > CustomElementBindingMap;
	148     OwnPtr<CustomElementBindingMap> m_customElementBindings;
	149 };

###init
	61 #define V8_STORE_PRIMORDIAL(name, Name) \
	62 { \
	63     ASSERT(m_##name##Prototype.isEmpty()); \
	64     v8::Handle<v8::String> symbol = v8::String::NewSymbol(#Name); \
	65     if (symbol.IsEmpty()) \
	66         return false; \
	67     v8::Handle<v8::Object> object = v8::Handle<v8::Object>::Cast(v8::Local<v8::Context>::New(m_isolate, m_context)->Global()->Get(symbol)); \
	68     if (object.IsEmpty()) \
	69         return false; \
	70     v8::Handle<v8::Value> prototypeValue = object->Get(prototypeString); \
	71     if (prototypeValue.IsEmpty()) \
	72         return false; \
	73     m_##name##Prototype.set(m_isolate, prototypeValue);  \
	74 }
	75
	76 bool V8PerContextData::init()
	77 {
	78     v8::Handle<v8::Context> context = v8::Local<v8::Context>::New(m_isolate, m_context);
	79     context->SetAlignedPointerInEmbedderData(v8ContextPerContextDataIndex, this);
	80
	81     v8::Handle<v8::String> prototypeString = v8::String::NewSymbol("prototype");
	82     if (prototypeString.IsEmpty())
	83         return false;
	84
	85     V8_STORE_PRIMORDIAL(error, Error);
	86
	87     return true;
	88 }
	90 #undef V8_STORE_PRIMORDIAL

###dispose
	49 void V8PerContextData::dispose()
	50 {
	51     v8::HandleScope handleScope(m_isolate);
	52     v8::Local<v8::Context>::New(m_isolate, m_context)->SetAlignedPointerInEmbedderData(v8ContextPerContextDataIndex, 0);
	53
	54     disposeMapWithUnsafePersistentValues(&m_wrapperBoilerplates);
	55     disposeMapWithUnsafePersistentValues(&m_constructorMap);
	56     m_customElementBindings.clear();
	57
	58     m_context.Dispose(); // persist object to dispose.
	59 }

## DOMWrapperWorld
###Class Define
This class represent a collection of DOM wrappers for a specific world.
	49 class DOMWrapperWorld : public RefCounted<DOMWrapperWorld> {
	50 public:
	51     static const int mainWorldId = 0;
	52     static const int mainWorldExtensionGroup = 0;
	53     static PassRefPtr<DOMWrapperWorld> ensureIsolatedWorld(int worldId, int extensionGroup);
	54     ~DOMWrapperWorld();
	55
	56     static bool isolatedWorldsExist() { return isolatedWorldCount; }
	57     static bool isIsolatedWorldId(int worldId) { return worldId > mainWorldId; }
	58     static void getAllWorlds(Vector<RefPtr<DOMWrapperWorld> >& worlds);
	59
	60     void setIsolatedWorldField(v8::Handle<v8::Context>);
	61
	62     static DOMWrapperWorld* isolatedWorld(v8::Handle<v8::Context> context)
	63     {
	64         ASSERT(contextHasCorrectPrototype(context));
	65         return static_cast<DOMWrapperWorld*>(context->GetAlignedPointerFromEmbedderData(v8ContextIsolatedWorld));
	66     }
	........................................
	108 private:
	109     static int isolatedWorldCount;
	110     static PassRefPtr<DOMWrapperWorld> createMainWorld();
	111     static bool contextHasCorrectPrototype(v8::Handle<v8::Context>);
	112
	113     DOMWrapperWorld(int worldId, int extensionGroup);
	114
	115     const int m_worldId;
	116     const int m_extensionGroup;
	117     OwnPtr<DOMDataStore> m_domDataStore;
	118
	119     friend DOMWrapperWorld* mainThreadNormalWorld();
	120     friend DOMWrapperWorld* existingWindowShellWorkaroundWorld();
	121 };
	122
	123 DOMWrapperWorld* mainThreadNormalWorld();

###mainThreadNormalWorld
	55 PassRefPtr<DOMWrapperWorld> DOMWrapperWorld::createMainWorld()
	56 {
	57     return adoptRef(new DOMWrapperWorld(mainWorldId, mainWorldExtensionGroup));
	58 }
	59
	60 DOMWrapperWorld::DOMWrapperWorld(int worldId, int extensionGroup)
	61     : m_worldId(worldId)
	62     , m_extensionGroup(extensionGroup)
	63 {
	64     if (isIsolatedWorld())
	65         m_domDataStore = adoptPtr(new DOMDataStore(IsolatedWorld));
	66 }
	80 DOMWrapperWorld* mainThreadNormalWorld() // get static local main DOMWrapperWorld.
	81 {
	82     ASSERT(isMainThread());
	83     DEFINE_STATIC_LOCAL(RefPtr<DOMWrapperWorld>, cachedNormalWorld, (DOMWrapperWorld::createMainWorld()));
	84     return cachedNormalWorld.get();
	85 }

###ensureIsolatedWorld
	107 typedef HashMap<int, DOMWrapperWorld*> WorldMap;
	108 static WorldMap& isolatedWorldMap()
	109 {
	110     ASSERT(isMainThread());
	111     DEFINE_STATIC_LOCAL(WorldMap, map, ());
	112     return map;
	113 }
	
	143 PassRefPtr<DOMWrapperWorld> DOMWrapperWorld::ensureIsolatedWorld(int worldId, int extensionGroup)
	144 {
	145     ASSERT(worldId > mainWorldId);
	146
	147     WorldMap& map = isolatedWorldMap();
	148     WorldMap::AddResult result = map.add(worldId, 0);
	149     RefPtr<DOMWrapperWorld> world = result.iterator->value;
	150     if (world) {
	151         ASSERT(world->worldId() == worldId);
	152         ASSERT(world->extensionGroup() == extensionGroup);
	153         return world.release();
	154     }
	155
	156     world = adoptRef(new DOMWrapperWorld(worldId, extensionGroup));
	157     result.iterator->value = world.get();
	158     isolatedWorldCount++;
	159     ASSERT(map.size() == isolatedWorldCount); //int DOMWrapperWorld::isolatedWorldCount = 0; global value.
	160
	161     return world.release();
	162 }

## ScriptController
###Class Define
	72 class ScriptController {
	73 public:
	74     ScriptController(Frame*);
	75     ~ScriptController();
	76
	77     bool initializeMainWorld();
	78     V8WindowShell* windowShell(DOMWrapperWorld*);
	79     V8WindowShell* existingWindowShell(DOMWrapperWorld*);
	81     ScriptValue executeScript(const ScriptSourceCode&);
	82     ScriptValue executeScript(const String& script, bool forceUserGesture = false);
	83
	84     // Evaluate JavaScript in the main world.
	85     ScriptValue executeScriptInMainWorld(const ScriptSourceCode&, AccessControlStatus = NotSharableCrossOrigin);
	86
	87     // Executes JavaScript in an isolated world. The script gets its own global scope,
	88     // its own prototypes for intrinsic JavaScript objects (String, Array, and so-on),
	89     // and its own wrappers for all DOM nodes and DOM constructors.
	90     //
	91     // If an isolated world with the specified ID already exists, it is reused.
	92     // Otherwise, a new world is created.
	93     //
	94     // FIXME: Get rid of extensionGroup here.
	95     void executeScriptInIsolatedWorld(int worldID, const Vector<ScriptSourceCode>& sources, int extensionGroup, Vector<ScriptValue>* results);
	96
	97     // Returns true if argument is a JavaScript URL.
	98     bool executeScriptIfJavaScriptURL(const KURL&);
	99
	100     v8::Local<v8::Value> compileAndRunScript(const ScriptSourceCode&, AccessControlStatus = NotSharableCrossOrigin);
	101
	102     v8::Local<v8::Value> callFunction(v8::Handle<v8::Function>, v8::Handle<v8::Object>, int argc, v8::Handle<v8::Value> argv[]);
	103     ScriptValue callFunctionEvenIfScriptDisabled(v8::Handle<v8::Function>, v8::Handle<v8::Object>, int argc, v8::Handle<v8::Value> argv[]);
	104     static v8::Local<v8::Value> callFunctionWithInstrumentation(ScriptExecutionContext*, v8::Handle<v8::Function>, v8::Handle<v8::Object> receiver, int argc, v8::Handle<v8::Value> args[]);
	105
	111     // Creates a property of the global object of a frame.
	112     void bindToWindowObject(Frame*, const String& key, NPObject*);
	113 ...................................................
	126     // Returns V8 Context. If none exists, creates a new context.
	127     // It is potentially slow and consumes memory.
	128     static v8::Local<v8::Context> mainWorldContext(Frame*);
	129     v8::Local<v8::Context> mainWorldContext();
	130     v8::Local<v8::Context> currentWorldContext();
	141     void clearWindowShell();
	142     void updateDocument();
	147     void updateSecurityOrigin();
	148     void clearScriptObjects();
	149     void cleanupScriptObjectsForPlugin(Widget*);
	150
	151     void clearForClose();
	152     void clearForOutOfMemory();
	163     bool setContextDebugId(int);
	164     static int contextDebugId(v8::Handle<v8::Context>);
	165
	166 private:
	167     typedef HashMap<int, OwnPtr<V8WindowShell> > IsolatedWorldMap;
	168     typedef HashMap<Widget*, NPObject*> PluginObjectMap;
	169
	170     void clearForClose(bool destroyGlobal);
	171
	172     Frame* m_frame;
	173     const String* m_sourceURL;
	174     v8::Isolate* m_isolate;
	175
	176     OwnPtr<V8WindowShell> m_windowShell; // has main windowShell
	177     IsolatedWorldMap m_isolatedWorlds; // a map of V8WindowShell object for different worldID.
	178
	179     bool m_paused;
	180 .........................................
	188 };

###Construct Function
	90 ScriptController::ScriptController(Frame* frame)
	91     : m_frame(frame)
	92     , m_sourceURL(0)
	93     , m_isolate(v8::Isolate::GetCurrent())
	94     , m_windowShell(V8WindowShell::create(frame, mainThreadNormalWorld(), m_isolate)) // when ScriptController is created, it'll create its main V8WindowShell.
	95     , m_paused(false)
	96     , m_windowScriptNPObject(0)
	97 {
	98 }

###windowShell
It'll get/create a V8WindowShell for a DOMWrapperWorld.
	66 V8WindowShell* ScriptController::windowShell(DOMWrapperWorld* world)
	67 {
	68     ASSERT(world);
	69
	70     V8WindowShell* shell = 0;
	71     if (world->isMainWorld())
	72         shell = m_windowShell.get();
	73     else {
	74         IsolatedWorldMap::iterator iter = m_isolatedWorlds.find(world->worldId());
	75         if (iter != m_isolatedWorlds.end())
	76             shell = iter->value.get();
	77         else {
	78             OwnPtr<V8WindowShell> isolatedWorldShell = V8WindowShell::create(m_frame, world, m_isolate);// create V8WindowShell.
	79             shell = isolatedWorldShell.get();
	80             m_isolatedWorlds.set(world->worldId(), isolatedWorldShell.release());
	81         }
	82     }
	83     if (!shell->isContextInitialized() && shell->initializeIfNeeded()) { // may create v8::Context.
	84         if (world->isMainWorld()) {
	85             // FIXME: Remove this if clause. See comment with existingWindowShellWorkaroundWorld().
	86             m_frame->loader()->dispatchDidClearWindowObjectInWorld(existingWindowShellWorkaroundWorld());
	87         } else
	88             m_frame->loader()->dispatchDidClearWindowObjectInWorld(world);
	89     }
	90     return shell;
	91 }

###updateDocument
	560 void ScriptController::updateDocument()
	561 {
	562     // For an uninitialized main window shell, do not incur the cost of context initialization during FrameLoader::init().
	563     if ((!m_windowShell->isContextInitialized() || !m_windowShell->isGlobalInitialized()) && m_frame->loader()->stateMachine()->creatingInitialEmptyDocument())
	564         return;
	565
	566     if (!initializeMainWorld())
	567         windowShell(mainThreadNormalWorld())->updateDocument();
	568 }
	
	###ScriptController::initializeMainWorld
	242 bool ScriptController::initializeMainWorld()
	243 {
	244     if (m_windowShell->isContextInitialized())
	245         return false;
	246     return windowShell(mainThreadNormalWorld())->isContextInitialized();
	247 }
	
	###ScriptController::clearForClose
	130 void ScriptController::clearForClose(bool destroyGlobal)
	131 {
	132     m_windowShell->clearForClose(destroyGlobal);
	133     for (IsolatedWorldMap::iterator iter = m_isolatedWorlds.begin(); iter != m_isolatedWorlds.end(); ++iter)
	134         iter->value->clearForClose(destroyGlobal);
	135     V8GCController::hintForCollectGarbage();
	136 }
	
	138 void ScriptController::clearForClose()
	139 {
	140     double start = currentTime();
	141     clearForClose(false);
	142     HistogramSupport::histogramCustomCounts("WebCore.ScriptController.clearForClose", (currentTime() - start) * 1000, 0, 10000, 50);
	143 }

## Frame have ScriptController Object.
###Frame Class Define
	70     class Frame : public RefCounted<Frame> {
	71     public:
	72         static PassRefPtr<Frame> create(Page*, HTMLFrameOwnerElement*, FrameLoaderClient*);
	73
	74         void init();
	75         void setView(PassRefPtr<FrameView>);
	76         void createView(const IntSize&, const StyleColor&, bool,
	77             const IntSize& fixedLayoutSize = IntSize(), bool useFixedLayout = false, ScrollbarMode = ScrollbarAuto, bool horizontalLock = false,
	78             ScrollbarMode = ScrollbarAuto, bool verticalLock = false);
	79
	80         ~Frame();
	81
	82         void addDestructionObserver(FrameDestructionObserver*);
	.....................................................
	222     inline ScriptController* Frame::script()
	223     {
	224         return m_script.get();
	225     }
	................................................
	171     private:
	172         Frame(Page*, HTMLFrameOwnerElement*, FrameLoaderClient*);
	173
	174         HashSet<FrameDestructionObserver*> m_destructionObservers;
	175
	176         Page* m_page;
	177         mutable FrameTree m_treeNode;
	178         mutable FrameLoader m_loader;
	179         mutable NavigationScheduler m_navigationScheduler;
	180
	181         HTMLFrameOwnerElement* m_ownerElement;
	182         RefPtr<FrameView> m_view;
	183         RefPtr<DOMWindow> m_domWindow;
	184
	185         OwnPtr<ScriptController> m_script; // play here.
	186         OwnPtr<Editor> m_editor;
	187         OwnPtr<FrameSelection> m_selection;
	188         OwnPtr<EventHandler> m_eventHandler;
	189         OwnPtr<AnimationController> m_animationController;
	190         OwnPtr<InputMethodController> m_inputMethodController;
	191
	192         float m_pageZoomFactor;
	193         float m_textZoomFactor;
	194
	195 #if ENABLE(ORIENTATION_EVENTS)
	196         int m_orientation;
	197 #endif
	198
	199         bool m_inViewSourceMode;
	200     };

###Frame Construct
	100 inline Frame::Frame(Page* page, HTMLFrameOwnerElement* ownerElement, FrameLoaderClient* frameLoaderClient)
	101     : m_page(page)
	102     , m_treeNode(this, parentFromOwnerElement(ownerElement))
	103     , m_loader(this, frameLoaderClient)
	104     , m_navigationScheduler(this)
	105     , m_ownerElement(ownerElement)
	106     , m_script(adoptPtr(new ScriptController(this))) // create ScriptController.
	107     , m_editor(adoptPtr(new Editor(this)))
	108     , m_selection(adoptPtr(new FrameSelection(this)))
	109     , m_eventHandler(adoptPtr(new EventHandler(this)))
	110     , m_animationController(adoptPtr(new AnimationController(this)))
	111     , m_inputMethodController(InputMethodController::create(this))
	112     , m_pageZoomFactor(parentPageZoomFactor(this))
	113     , m_textZoomFactor(parentTextZoomFactor(this))
	114 #if ENABLE(ORIENTATION_EVENTS)
	115     , m_orientation(0)
	116 #endif
	117     , m_inViewSourceMode(false)
	118 {
	119     ASSERT(page);
	120
	121     if (ownerElement) {
	122         page->incrementSubframeCount();
	123         ownerElement->setContentFrame(this);
	124     }
	125
	126 #ifndef NDEBUG
	127     frameCounter.increment();
	128 #endif
	129 }

## Who Call initializeMainWorld
->ScriptController::updateDocument()=>ScriptController::initializeMainWorld()=>ScriptController::windowShell=>V8WindowShell::create/initializeIfNeeded

###DocumentLoader::createWriterFor
	991 PassRefPtr<DocumentWriter> DocumentLoader::createWriterFor(Frame* frame, const Document* ownerDocument, const KURL& url, const String& mimeType, const String& encoding, bool userChosen, bool dispatch)
	992 {
	993     // Create a new document before clearing the frame, because it may need to
	994     // inherit an aliased security context.
	995     RefPtr<Document> document = DOMImplementation::createDocument(mimeType, frame, url, frame->inViewSourceMode());
	996     if (document->isPluginDocument() && document->isSandboxed(SandboxPlugins))
	997         document = SinkDocument::create(DocumentInit(url, frame));
	998     bool shouldReuseDefaultView = frame->loader()->stateMachine()->isDisplayingInitialEmptyDocument() && frame->document()->isSecureTransitionTo(url);
	999
	1000     ClearOptions options = 0;
	1001     if (!shouldReuseDefaultView)
	1002         options = ClearWindowProperties | ClearScriptObjects;
	1003     frame->loader()->clear(options);
	1004
	1005     if (frame->document() && frame->document()->attached())
	1006         frame->document()->prepareForDestruction();
	1007
	1008     if (!shouldReuseDefaultView)
	1009         frame->setDOMWindow(DOMWindow::create(frame)); // it'll create DOMWindow
	1010
	1011     frame->loader()->setOutgoingReferrer(url);
	1012     frame->domWindow()->setDocument(document); // call to DOMWindow's setDocument
	1013
	1014     if (ownerDocument) {
	1015         document->setCookieURL(ownerDocument->cookieURL());
	1016         document->setSecurityOrigin(ownerDocument->securityOrigin());
	1017     }
	1018
	1019     frame->loader()->didBeginDocument(dispatch);
	1020
	1021     return DocumentWriter::create(document.get(), mimeType, encoding, userChosen);
	1022 }

###DOMWindow::setDocument
	325 void DOMWindow::setDocument(PassRefPtr<Document> document)
	326 {
	327     ASSERT(!document || document->frame() == m_frame);
	328     if (m_document) {
	329         if (m_document->attached()) {
	330             // FIXME: We don't call willRemove here. Why is that OK?
	331             // This detach() call is also mostly redundant. Most of the calls to
	332             // this function come via DocumentLoader::createWriterFor, which
	333             // always detaches the previous Document first. Only XSLTProcessor
	334             // depends on this detach() call, so it seems like there's some room
	335             // for cleanup.
	336             m_document->detach();
	337         }
	338         m_document->setDOMWindow(0);
	339     }
	340
	341     m_document = document;
	342
	343     if (!m_document)
	344         return;
	345
	346     m_document->setDOMWindow(this);
	347     if (!m_document->attached())
	348         m_document->attach();
	349
	350     if (!m_frame)
	351         return;
	352
	353     m_frame->script()->updateDocument(); // it's important for its time to ScriptController's V8WindowShell.
	354     m_document->updateViewportArguments();
	355
	356     if (m_frame->page() && m_frame->view()) {
	357         if (ScrollingCoordinator* scrollingCoordinator = m_frame->page()->scrollingCoordinator()) {
	358             scrollingCoordinator->scrollableAreaScrollbarLayerDidChange(m_frame->view(), HorizontalScrollbar);
	359             scrollingCoordinator->scrollableAreaScrollbarLayerDidChange(m_frame->view(), VerticalScrollbar);
	360             scrollingCoordinator->scrollableAreaScrollLayerDidChange(m_frame->view());
	361         }
	362     }
	363
	364     m_frame->selection()->updateSecureKeyboardEntryIfActive();
	365
	366     if (m_frame->page() && m_frame->page()->mainFrame() == m_frame) {
	367         m_frame->page()->mainFrame()->notifyChromeClientWheelEventHandlerCountChanged();
	368         if (m_document->hasTouchEventHandlers())
	369             m_frame->page()->chrome().client()->needTouchEvents(true);
	370     }
	371 }

##How about V8Window&&V8Window::info
See Next.