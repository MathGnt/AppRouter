# AppRouter - Architecture et Fonctionnement

## Vue d'ensemble

AppRouter est une bibliothèque de navigation SwiftUI basée sur des protocoles génériques qui permet de gérer :
- Navigation simple (single stack)
- Navigation à onglets (tab-based) avec stacks indépendants
- Présentation de sheets
- Deep linking via URLs

## Les 3 Protocoles Fondamentaux

### 1. `DestinationType`
```swift
public protocol DestinationType: Hashable {
  static func from(path: String, fullPath: [String], parameters: [String: String]) -> Self?
}
```

**Rôle** : Représente une destination de navigation dans un NavigationStack

**Utilisation** :
```swift
enum Destination: DestinationType {
    case home
    case detail(id: String)
    case profile(userId: String)

    static func from(path: String, fullPath: [String], parameters: [String: String]) -> Self? {
        switch path {
        case "home": return .home
        case "detail": return .detail(id: parameters["id"] ?? "")
        case "profile": return .profile(userId: parameters["userId"] ?? "")
        default: return nil
        }
    }
}
```

**Contraintes** : Doit être `Hashable` pour fonctionner avec NavigationStack

---

### 2. `SheetType`
```swift
public protocol SheetType: Hashable, Identifiable {}
```

**Rôle** : Représente une sheet (modal) présentable

**Utilisation** :
```swift
enum Sheet: SheetType {
    case settings
    case editProfile(userId: String)
    case createItem

    var id: String {
        switch self {
        case .settings: return "settings"
        case .editProfile(let id): return "editProfile-\(id)"
        case .createItem: return "createItem"
        }
    }
}
```

**Contraintes** :
- `Hashable` pour égalité
- `Identifiable` pour binding SwiftUI (`.sheet(item: $router.presentedSheet)`)

---

### 3. `TabType`
```swift
public protocol TabType: Hashable, CaseIterable, Identifiable, Sendable {
  var icon: String { get }
}
```

**Rôle** : Représente un onglet dans un TabView (uniquement pour Router, pas SimpleRouter)

**Utilisation** :
```swift
enum AppTab: String, TabType {
    case home
    case search
    case profile

    var id: String { rawValue }

    var icon: String {
        switch self {
        case .home: return "house"
        case .search: return "magnifyingglass"
        case .profile: return "person"
        }
    }
}
```

**Contraintes** :
- `CaseIterable` : pour itérer sur tous les onglets
- `Identifiable` : pour identification SwiftUI
- `Sendable` : pour concurrence thread-safe

---

## Les 2 Classes Router

### SimpleRouter<Destination, Sheet>

**Pour qui** : Apps avec navigation simple (1 seul NavigationStack)

**Structure** :
```swift
@Observable @MainActor
public final class SimpleRouter<Destination: DestinationType, Sheet: SheetType> {
    public var path: [Destination] = []           // Stack de navigation
    public var presentedSheet: Sheet?             // Sheet actuelle
}
```

**Méthodes clés** :
- `navigateTo(_ destination: Destination)` - Push une destination
- `popNavigation()` - Pop la dernière destination
- `popToRoot()` - Retour à la racine
- `presentSheet(_ sheet: Sheet)` - Affiche une sheet
- `dismissSheet()` - Ferme la sheet
- `navigate(to: URL)` - Deep linking

**Usage** :
```swift
@State private var router = SimpleRouter<Destination, Sheet>()

NavigationStack(path: $router.path) {
    RootView()
}
.sheet(item: $router.presentedSheet) { sheet in
    SheetView(sheet: sheet)
}
.environment(router)
```

---

### Router<Tab, Destination, Sheet>

**Pour qui** : Apps avec TabView et navigation indépendante par onglet

**Structure** :
```swift
@Observable @MainActor
public final class Router<Tab: TabType, Destination: DestinationType, Sheet: SheetType> {
    private var paths: [Tab: [Destination]] = [:] // Stack par onglet
    public var sheetPaths: [Sheet] = []           // Stack de sheets imbriquées
    public var selectedTab: Tab                   // Onglet actif
    public var presentedSheet: Sheet?             // Sheet actuelle
}
```

**Méthodes clés** :
- `navigateTo(_ destination: Destination, for tab: Tab? = nil)` - Push dans un onglet
- `popNavigation(for tab: Tab? = nil)` - Pop dans un onglet
- `popToNavRoot(for tab: Tab? = nil)` - Retour root d'un onglet
- `popToSheetRoot()` - Vide le stack de sheets
- `popToAllRoots(for tab: Tab? = nil)` - Reset complet
- `subscript(tab: Tab) -> [Destination]` - Accès au path d'un onglet

**Usage** :
```swift
@State private var router = Router<AppTab, Destination, Sheet>(initialTab: .home)

TabView(selection: $router.selectedTab) {
    ForEach(AppTab.allCases) { tab in
        NavigationStack(path: $router[tab]) {
            TabRootView(tab: tab)
        }
        .tabItem { Label(tab.rawValue, systemImage: tab.icon) }
        .tag(tab)
    }
}
.sheet(item: $router.presentedSheet) { sheet in
    NavigationStack(path: $router.sheetPaths) {
        SheetView(sheet: sheet)
    }
}
.environment(router)
```

---

## Pattern d'Utilisation

### 1. Définir vos types

```swift
// Vos destinations
enum Destination: DestinationType {
    case home
    case detail(id: String)

    static func from(path: String, fullPath: [String], parameters: [String: String]) -> Self? {
        // Parsing pour deep linking
    }
}

// Vos sheets
enum Sheet: SheetType {
    case settings
    case edit(id: String)
    var id: String { /* ... */ }
}

// Vos onglets (si nécessaire)
enum AppTab: String, TabType {
    case home, profile
    var id: String { rawValue }
    var icon: String { /* ... */ }
}
```

### 2. Créer un typealias (recommandé)

```swift
// Pour SimpleRouter
typealias AppRouter = SimpleRouter<Destination, Sheet>

// Pour Router
typealias AppRouter = Router<AppTab, Destination, Sheet>
```

### 3. Injecter dans l'environnement

```swift
@State private var router = AppRouter(/* ... */)

NavigationStack(/* ... */)
    .environment(router)
```

### 4. Utiliser dans les vues enfants

```swift
struct ContentView: View {
    @Environment(AppRouter.self) private var router

    var body: some View {
        Button("Go to detail") {
            router.navigateTo(.detail(id: "123"))
        }
        Button("Open settings") {
            router.presentSheet(.settings)
        }
    }
}
```

---

## Deep Linking (URLs)

Le protocole `DestinationType` permet de parser des URLs :

```swift
// URL: myapp://detail/profile?userId=123
router.navigate(to: "myapp://detail/profile?userId=123")

// Appelle:
Destination.from(path: "detail", fullPath: ["detail", "profile"], parameters: ["userId": "123"])
Destination.from(path: "profile", fullPath: ["detail", "profile"], parameters: ["userId": "123"])
```

Le router reconstruit le stack de navigation complet depuis l'URL.

---

## Avantages du Design par Protocoles

1. **Type-safe** : Compile-time checking des destinations/sheets
2. **Flexible** : Définissez vos propres types (enum, struct, etc.)
3. **Testable** : Mockable facilement
4. **Générique** : Fonctionne avec n'importe quels types conformes
5. **SwiftUI-native** : Utilise @Observable et @MainActor
6. **Pas de dépendances** : Pure Swift/SwiftUI

---

## Limitations Actuelles

1. **Verbosité des types** : `Router<Tab, Destination, Sheet>` est long
   - **Solution** : Utiliser `typealias AppRouter = Router<...>`

2. **@Environment répétitif** : `@Environment(AppRouter.self) private var router`
   - **Solution possible** : EnvironmentKey custom ou property wrapper

3. **Pas de type erasure** : Impossible d'avoir un protocole commun entre SimpleRouter et Router sans perdre la type-safety
