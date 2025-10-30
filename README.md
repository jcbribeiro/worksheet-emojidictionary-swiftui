## SwiftUI Practical Worksheet: Building an Emoji Dictionary App

This worksheet is the SwiftUI version of the **Emoji Dictionary** you built with **storyboards / UIKit**. The goal is to build a similar app, but in a **declarative** way, learning the SwiftUI equivalents of what you already know.

### 0. Project Setup

1. Open **Xcode** → **File > New > Project…**
2. Template: **iOS > App**
3. Name: **EmojiDictionarySwiftUI**
4. Interface: **SwiftUI**
5. Language: **Swift**

Xcode will create:

* `EmojiDictionarySwiftUIApp.swift` (entry point)
* `ContentView.swift` (initial view)

> #### SwiftUI vs Storyboard
> * **Storyboard / UIKit**: you design screens visually, then connect outlets and actions.
> * **SwiftUI**: you **describe** the UI in code; the UI is a *function of state*.
> * Key idea: **when data changes, UI recomputes**.
> 
> Docs: 
> * Introducing SwiftUI — Apple Developer - [https://developer.apple.com/xcode/swiftui/](https://developer.apple.com/xcode/swiftui/)
> * Apple SwiftUI Tutorials - [https://developer.apple.com/tutorials/swiftui](https://developer.apple.com/tutorials/swiftui)


## 1. Model

Create a new file **Emoji.swift**.

```swift
import Foundation
import Combine

struct Emoji: Codable, Identifiable {
    var id = UUID()
    var symbol: String
    var name: String
    var description: String
    var usage: String
    
    // Location of the plist file
    static var archiveURL: URL {
        let documentsURL = FileManager.default
            .urls(for: .documentDirectory, in: .userDomainMask)
            .first!
        return documentsURL
            .appendingPathComponent("emojis")
            .appendingPathExtension("plist")
    }
    
    static var sampleEmojis: [Emoji] {
        [
            Emoji(symbol: "😀", name: "Grinning Face", description: "A typical smiley face.", usage: "happiness"),
            Emoji(symbol: "😕", name: "Confused Face", description: "A puzzled face.", usage: "unsure"),
            Emoji(symbol: "😍", name: "Heart Eyes", description: "Loves something.", usage: "love"),
            Emoji(symbol: "🧑‍💻", name: "Developer", description: "Working on Mac.", usage: "software, coding"),
            Emoji(symbol: "🐢", name: "Turtle", description: "Slow but steady.", usage: "slow"),
            Emoji(symbol: "📚", name: "Books", description: "Stack of books.", usage: "study")
        ]
    }
    
    // Save to plist
    static func saveToFile(emojis: [Emoji]) {
        let encoder = PropertyListEncoder()
        do {
            let data = try encoder.encode(emojis)
            try data.write(to: archiveURL)
        } catch {
            print("Error encoding emojis: \(error)")
        }
    }
    
    // Load from plist
    static func loadFromFile() -> [Emoji]? {
        guard let data = try? Data(contentsOf: archiveURL) else {
            return nil
        }
        do {
            let decoder = PropertyListDecoder()
            return try decoder.decode([Emoji].self, from: data)
        } catch {
            print("Error decoding emojis: \(error)")
            return nil
        }
    }
}

// MARK: - Preview data
#if DEBUG
extension Emoji {
    static var preview = Emoji(symbol: "🧪", name: "Test", description: "Preview emoji", usage: "debug")
}
#endif
```

* Data is saved to a **plist** in the app’s documents folder. This is not a replacement for Core Data / SwiftData.

### 1.1 Observable Data Source

In the same file (or a new one, e.g. `EmojiData.swift`):

```swift
class EmojiData: ObservableObject {
    @Published var emojis: [Emoji] = [] {
        didSet {
            Emoji.saveToFile(emojis: emojis)
        }
    }
    
    init() {
        if let saved = Emoji.loadFromFile() {
            emojis = saved
        } else {
            emojis = Emoji.sampleEmojis
        }
    }
}
```

> #### `ObservableObject`, `@Published`, `@StateObject`
> * `ObservableObject`: a class that can be **observed** by SwiftUI views. When an observed property of this class changes, SwiftUI knows it should **re-render any view** that depends on it. Here, `EmojiData` acts as the **shared data source** for the app.
> * `@Published var emojis`: a property wrapper that automatically announces (publishes) changes. When a `@Published` variable is modified, all views observing the object are updated. Here, whenever `emojis` changes (add, delete, reorder), SwiftUI **refreshes** the list view automatically — no manual reloads needed.
> * `@StateObject`: used **in the view that creates** the object. SwiftUI ensures the object stays alive **for the lifetime of that view**, even if the view’s body is re-rendered. This is typically placed in your **root or top-level view**.
> * `@ObservedObject`: used **in child views** that receive the object. The child doesn’t own it — it just observes updates. If `emojiData.emojis` changes in the parent, the child’s UI updates automatically.
> * `@State` declares a **piece of local, view-owned data**. It is **private** to the view and automatically stored by SwiftUI. When its value changes (e.g. from `false` to `true`), SwiftUI **updates the view hierarchy** that depends on it.
>
> Docs:
> * [https://developer.apple.com/documentation/combine/observableobject](https://developer.apple.com/documentation/combine/observableobject)
> * [https://developer.apple.com/documentation/swiftui/stateobject](https://developer.apple.com/documentation/swiftui/stateobject)


## 2. List Screen (Main Screen)

Edit **ContentView.swift** to the version below.

```swift
import SwiftUI

struct ContentView: View {
    @StateObject private var emojiData = EmojiData()
    @State private var showingAddEmoji = false
    
    var body: some View {
        NavigationStack {
            List {
                // Binding-based ForEach: we get a Binding<Emoji> for each item
                ForEach($emojiData.emojis) { $emoji in
                    NavigationLink {
                        AddEditEmojiView(emoji: $emoji, isEditing: true)
                    } label: {
                        EmojiRowView(emoji: emoji)
                    }
                }
                .onDelete { indexSet in
                    emojiData.emojis.remove(atOffsets: indexSet)
                }
                .onMove { indices, newOffset in
                    emojiData.emojis.move(fromOffsets: indices, toOffset: newOffset)
                }
            }
            .navigationTitle("Emojis")
            .toolbar {
                ToolbarItem(placement: .navigationBarLeading) {
                    EditButton()
                }
                ToolbarItem(placement: .navigationBarTrailing) {
                    Button {
                        showingAddEmoji = true
                    } label: {
                        Image(systemName: "plus")
                    }
                    .accessibilityLabel("Add Emoji")
                }
            }
            .sheet(isPresented: $showingAddEmoji) {
                AddEditEmojiView(
                    emoji: .constant(Emoji(symbol: "", name: "", description: "", usage: "")),
                    isEditing: false
                ) { newEmoji in
                    emojiData.emojis.append(newEmoji)
                }
            }
        }
    }
}

// MARK: - Preview
#Preview {
    ContentView()
}
```


`.onDelete`, `.onMove` and `EditButton()` are the SwiftUI equivalent of UITableView’s **editing style**.

> #### Binding (`$`)
> * In SwiftUI, **data flows down**, **bindings go up**. When we write `$emojiData.emojis`, we are not passing the value, we are passing a **connection to the value**.
> * The `$` symbol turns a **value** into a **binding**, a live connection to the original data - so edits in the child view update the array.
> 
> With
> ```swift
> ForEach($emojiData.emojis) { $emoji in ... }
> ```
> we get a binding to each element: we can pass it to `AddEditEmojiView`; changes are written back to the array; `@Published` fires; UI updates; file is saved.
>
> This is the **“data drives UI”** pattern. This pattern embodies the SwiftUI philosophy: “Your views don’t control the data — the data controls the views.”
> 
> Docs: [https://developer.apple.com/documentation/swiftui/binding](https://developer.apple.com/documentation/swiftui/binding)

## 3. Row View

Create **EmojiRowView.swift** (or put under ContentView for simplicity):

```swift
import SwiftUI

struct EmojiRowView: View {
    let emoji: Emoji
    
    var body: some View {
        HStack {
            Text(emoji.symbol)
                .font(.largeTitle)
            VStack(alignment: .leading) {
                Text(emoji.name)
                    .font(.headline)
                Text(emoji.description)
                    .font(.caption)
                    .foregroundStyle(.secondary)
            }
        }
        .padding(.vertical, 4)
    }
}

#Preview {
    EmojiRowView(emoji: .preview)
        .previewLayout(.sizeThatFits)
        .padding()
}
```

This is equivalent to configuring the prototype cell in the storyboard.

## 4. Add / Edit View (Form)

Create **AddEditEmojiView.swift** (or put under ContentView for simplicity).

```swift
import SwiftUI

struct AddEditEmojiView: View {
    @Binding var emoji: Emoji
    var isEditing: Bool
    var onSave: ((Emoji) -> Void)? = nil
    
    @Environment(\.dismiss) private var dismiss
    @State private var showAlert = false
    
    var body: some View {
        NavigationStack {
            Form {
                Section("Symbol") {
                    TextField("The emoji", text: $emoji.symbol)
                        .textInputAutocapitalization(.never)
                }
                
                Section("Name") {
                    TextField("The emoji's description", text: $emoji.name)
                }
                
                Section("Description") {
                    TextField("Describe this emoji", text: $emoji.description)
                }
                
                Section("Usage") {
                    TextField("When do you use it?", text: $emoji.usage)
                }
            }
            .navigationTitle(isEditing ? "Edit Emoji" : "New Emoji")
            .toolbar {
                ToolbarItem(placement: .cancellationAction) {
                    Button("Cancel") {
                        dismiss()
                    }
                }
                ToolbarItem(placement: .confirmationAction) {
                    Button(isEditing ? "Save" : "Add") {
                        if let onSave {
                            onSave(emoji)
                            showAlert = true
                        } else {
                            // editing an existing emoji (binding updates the list directly)
                            dismiss()
                        }
                    }
                }
            }
            .alert("Emoji added!", isPresented: $showAlert) {
                Button("OK") { dismiss() }
            } message: {
                Text("Your emoji was added to the dictionary.")
            }
        }
    }
}

#Preview("Add Mode") {
    AddEditEmojiView(
        emoji: .constant(Emoji(symbol: "", name: "", description: "", usage: "")),
        isEditing: false
    ) { _ in }
}

#Preview("Edit Mode") {
    AddEditEmojiView(
        emoji: .constant(.preview),
        isEditing: true
    )
}
```

> #### `@Environment(\.dismiss)`
> * In storyboard you usually call `dismiss(animated:)` or pop the controller.
> * In SwiftUI, you ask the **environment** to dismiss the current presentation.
> * It works for both modals (`sheet`) and navigation.
> 
> Docs: [https://developer.apple.com/documentation/swiftui/environmentvalues/dismiss](https://developer.apple.com/documentation/swiftui/environmentvalues/dismiss)

> #### Forms in SwiftUI
> * `Form { ... }` automatically gives the grouped iOS form style, ideal for data entry. Similar to static table views in storyboard, but simpler.
> 
> Docs: [https://developer.apple.com/documentation/swiftui/form](https://developer.apple.com/documentation/swiftui/form)

## 5. App Entry Point

Edit the file Xcode created (e.g. `EmojiDictionarySwiftUIApp.swift`):

```swift
import SwiftUI

@main
struct EmojiDictionarySwiftUIApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
        }
    }
}
```

