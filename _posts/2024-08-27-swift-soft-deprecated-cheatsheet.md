---
tag: Code
title: Swift Soft-deprecated API Cheatsheet
date: 2024-08-27T01:09:04-0000
---

## Last Updated
2024-09-16T02:12:15-0000

## Swift

### Method
- `URL`
  - `appendingPathComponent(_ pathComponent: String) -> URL`
  - `appendingPathComponent(_ pathComponent: String, isDirectory: Bool) -> URL`
  - `appendPathComponent(_ pathComponent: String)`
  - `appendPathComponent(_ pathComponent: String, isDirectory: Bool)`
- `NSCoder`
  - `decodeTopLevelObject() throws -> Any?`
  - `decodeTopLevelObject(forKey key: String) throws -> Any?`

#### Initializer
- `URL`
  - `init(fileURLWithPath path: String)`
  - `init(fileURLWithPath path: String, isDirectory: Bool)`
  - `init(fileURLWithPath path: String, relativeTo base: URL?)`
  - `init(fileURLWithPath path: String, isDirectory: Bool, relativeTo base: URL?)`

### Property
- `URL`
  - `host`
  - `user`
  - `path`
  - `query`
  - `fragment`
  - `password`
- `URLComponents`
  - `percentEncodedHost`
- `URLResourceValues`
  - `typeIdentifier`

## SwiftUI

### Method
- `DropInfo.hasItemConforming(to types: [String]) -> Bool`
- `DropInfo.itemProviders(for types: [String]) -> [NSItemProvider]`
- `GeometryProxy.frame(in coordinateSpace: CoordinateSpace) -> CGRect`

#### Initializer
- Font
  - `Font.system(_ style: Font.TextStyle, design: Font.Design)`
  - `Font.system(size: CGFloat, weight: Font.Weight, design: Font.Design)`
- Gesture
  - `SpatialTapGesture.init(count: Int, coordinateSpace: CoordinateSpace)`
  - `DragGesture.Value.init(minimumDistance: CGFloat, coordinateSpace: CoordinateSpace)`
- View Style
  - `SwitchToggleStyle.init(tint: Color)`
  - `LinearProgressViewStyle.init(tint: Color)`
  - `CircularProgressViewStyle.init(tint: Color)`
- View
  - `Color`
    - `init(_ color: UIColor)`
    - `init(_ cgColor: CGColor)`
  - `DynamicViewContent`
    - `onInsert(of acceptedTypeIdentifiers: [String], perform action: @escaping (Int, [NSItemProvider]) -> Void)`
  - `GroupBox`
    - `init(label: Label, @ViewBuilder content: () -> Content)`
  - `NavigationLink`
    - `init(_ titleKey: LocalizedStringKey, destination: Destination)`
    - `init(destination: Destination, @ViewBuilder label: () -> Label)`
    - `init<S>(_ title: S, destination: Destination) where S : StringProtocol`
  - `Picker`
    - `init(selection: Binding<SelectionValue>, label: Label, @ViewBuilder content: () -> Content)`
  - `ScrollView`
    - `init(_ axes: Axis.Set, showsIndicators: Bool, @ViewBuilder content: () -> Content)`
  - `Section`
    - `init(header: Parent, @ViewBuilder content: () -> Content)`
    - `init(footer: Footer, @ViewBuilder content: () -> Content)`
    - `init(header: Parent, footer: Footer, @ViewBuilder content: () -> Content)`
  - `SecureField`
    - `init(_ titleKey: LocalizedStringKey, text: Binding<String>, onCommit: @escaping () -> Void)`
    - `init<S>(_ title: S, text: Binding<String>, onCommit: @escaping () -> Void) where S : StringProtocol`
  - `Slider`
    - `init<V>(value: Binding<V>, in bounds: ClosedRange<V>, onEditingChanged: @escaping (Bool) -> Void, @ViewBuilder label: () -> Label) where V : BinaryFloatingPoint, V.Stride : BinaryFloatingPoint`
    - `init<V>(value: Binding<V>, in bounds: ClosedRange<V>, onEditingChanged: @escaping (Bool) -> Void, minimumValueLabel: ValueLabel, maximumValueLabel: ValueLabel, @ViewBuilder label: () -> Label) where V : BinaryFloatingPoint, V.Stride : BinaryFloatingPoint`
    - `init<V>(value: Binding<V>, in bounds: ClosedRange<V>, step: V.Stride, onEditingChanged: @escaping (Bool) -> Void, @ViewBuilder label: () -> Label) where V : BinaryFloatingPoint, V.Stride : BinaryFloatingPoint`
    - `init<V>(value: Binding<V>, in bounds: ClosedRange<V>, step: V.Stride, onEditingChanged: @escaping (Bool) -> Void, minimumValueLabel: ValueLabel, maximumValueLabel: ValueLabel, @ViewBuilder label: () -> Label) where V : BinaryFloatingPoint, V.Stride : BinaryFloatingPoint`
  - `Stepper`
    - `init(onIncrement: (() -> Void)?, onDecrement: (() -> Void)?, onEditingChanged: @escaping (Bool) -> Void, @ViewBuilder label: () -> Label)`
    - `init<V>(value: Binding<V>, step: V.Stride, onEditingChanged: @escaping (Bool) -> Void, @ViewBuilder label: () -> Label) where V : Strideable`
    - `init<V>(value: Binding<V>, in bounds: ClosedRange<V>, step: V.Stride, onEditingChanged: @escaping (Bool) -> Void, @ViewBuilder label: () -> Label) where V : Strideable`
  - `TextField`
    - `init<S>(_ title: S, text: Binding<String>, onCommit: @escaping () -> Void) where S : StringProtocol`
    - `init<S>(_ title: S, text: Binding<String>, onEditingChanged: @escaping (Bool) -> Void) where S : StringProtocol`
    - `init<S>(_ title: S, text: Binding<String>, onEditingChanged: @escaping (Bool) -> Void, onCommit: @escaping () -> Void) where S : StringProtocol`
    - `init(_ titleKey: LocalizedStringKey, text: Binding<String>, onCommit: @escaping () -> Void)`
    - `init(_ titleKey: LocalizedStringKey, text: Binding<String>, onEditingChanged: @escaping (Bool) -> Void)`
    - `init(_ titleKey: LocalizedStringKey, text: Binding<String>, onEditingChanged: @escaping (Bool) -> Void, onCommit: @escaping () -> Void)`
    - `init<V>(_ titleKey: LocalizedStringKey, value: Binding<V>, formatter: Formatter, onCommit: @escaping () -> Void)`
    - `init<V>(_ titleKey: LocalizedStringKey, value: Binding<V>, formatter: Formatter, onEditingChanged: @escaping (Bool) -> Void)`
    - `init<V>(_ titleKey: LocalizedStringKey, value: Binding<V>, formatter: Formatter, onEditingChanged: @escaping (Bool) -> Void, onCommit: @escaping () -> Void)`
    - `init<S, V>(_ title: S, value: Binding<V>, formatter: Formatter, onCommit: @escaping () -> Void) where S : StringProtocol`
    - `init<S, V>(_ title: S, value: Binding<V>, formatter: Formatter, onEditingChanged: @escaping (Bool) -> Void) where S : StringProtocol`
    - `init<S, V>(_ title: S, value: Binding<V>, formatter: Formatter, onEditingChanged: @escaping (Bool) -> Void, onCommit: @escaping () -> Void) where S : StringProtocol`
  - `ToolbarItem`
    - `init(id: String, placement: ToolbarItemPlacement, showsByDefault: Bool, @ViewBuilder content: () -> Content)`

#### View Modifier
- `accentColor(_ accentColor: Color?)`
- `background<Background>(_ background: Background, alignment: Alignment) where Background : View`
- `colorScheme(_ colorScheme: ColorScheme)`
- `coordinateSpace<T>(name: T) where T : Hashable`
- `cornerRadius(_ radius: CGFloat, antialiased: Bool)`
- `foregroundColor(_ color: Color?)`
- `mask<Mask>(_ mask: Mask) where Mask : View`
- `navigationViewStyle<S>(_ style: S) where S : NavigationViewStyle`
- `overlay<Overlay>(_ overlay: Overlay, alignment: Alignment) where Overlay : View`
- Accessibility
  - `accessibility(hint: Text)`
  - `accessibility(label: Text)`
  - `accessibility(value: Text)`
  - `accessibility(hidden: Bool)`
  - `accessibility(identifier: String)`
  - `accessibility(inputLabels: [Text])`
  - `accessibility(sortPriority: Double)`
  - `accessibility(activationPoint: CGPoint)`
  - `accessibility(activationPoint: UnitPoint)`
  - `accessibility(selectionIdentifier: AnyHashable)`
  - `accessibility(addTraits traits: AccessibilityTraits)`
  - `accessibility(removeTraits traits: AccessibilityTraits)`
- Editing
  - `autocapitalization(_ style: UITextAutocapitalizationType)`
  - `disableAutocorrection(_ disable: Bool?)`
- Gesture
  - `onTapGesture(count: Int, coordinateSpace: CoordinateSpace, perform action: @escaping (CGPoint) -> Void)`
  - `onContinuousHover(coordinateSpace: CoordinateSpace, perform action: @escaping (HoverPhase) -> Void)`
  - `onLongPressGesture(minimumDuration: Double, maximumDistance: CGFloat, pressing: ((Bool) -> Void)?, perform action: @escaping () -> Void)`
  - `onDrop(of supportedTypes: [String], delegate: any DropDelegate)`
  - `onDrop(of supportedTypes: [String], isTargeted: Binding<Bool>?, perform action: @escaping (_ providers: [NSItemProvider]) -> Bool)`
  - `onDrop(of supportedTypes: [String], isTargeted: Binding<Bool>?, perform action: @escaping (_ providers: [NSItemProvider], _ location: CGPoint) -> Bool)`
- Navigation Bar
  - `navigationBarHidden(_ hidden: Bool)`
  - `navigationBarItems<L>(leading: L) where L : View`
  - `navigationBarItems<T>(trailing: T) where T : View`
  - `navigationBarItems<L, T>(leading: L, trailing: T) where L : View, T : View`
  - `navigationBarTitle(_ title: Text)`
  - `navigationBarTitle(_ titleKey: LocalizedStringKey)`
  - `navigationBarTitle<S>(_ title: S) where S : StringProtocol`
  - `navigationBarTitle(_ title: Text, displayMode: NavigationBarItem.TitleDisplayMode)`
  - `navigationBarTitle(_ titleKey: LocalizedStringKey, displayMode: NavigationBarItem.TitleDisplayMode)`
  - `navigationBarTitle<S>(_ title: S, displayMode: NavigationBarItem.TitleDisplayMode) where S : StringProtocol`
- Presentation
  - `actionSheet(isPresented: Binding<Bool>, content: () -> ActionSheet)`
  - `actionSheet<T>(item: Binding<T?>, content: (T) -> ActionSheet) where T : Identifiable`
  - `alert(isPresented: Binding<Bool>, content: () -> Alert)`
  - `alert<Item>(item: Binding<Item?>, content: (Item) -> Alert) where Item : Identifiable`
  - `contextMenu<MenuItems>(_ contextMenu: ContextMenu<MenuItems>?) where MenuItems : View`
- Searchable
  - `searchable<S>(text: Binding<String>, placement: SearchFieldPlacement, prompt: Text?, @ViewBuilder suggestions: () -> S) where S : View`
  - `searchable<S>(text: Binding<String>, placement: SearchFieldPlacement, prompt: LocalizedStringKey, @ViewBuilder suggestions: () -> S) where S : View`
  - `searchable<V, S>(text: Binding<String>, placement: SearchFieldPlacement, prompt: S, @ViewBuilder suggestions: () -> V) where V : View, S : StringProtocol`
- System Interface
  - `edgesIgnoringSafeArea(_ edges: Edge.Set)`
  - `statusBar(hidden: Bool)`

### Property
- `Color`
  - `cgColor`
- `EnvironmentValues`
  - `sizeCategory`
  - `presentationMode`
  - `controlActiveState`
  - `disableAutocorrection`

#### Static
- `MenuStyle`
  - `borderlessButton`
- `ControlActiveState`
  - `AllCases`
- `NavigationViewStyle`
  - `stack`
  - `columns`
  - `automatic`
- `ToolbarItemPlacement`
  - `navigationBarLeading`
  - `navigationBarTrailing`

### Type
- `Alert`
- `ActionSheet`
- `PresentationMode`
- `AnimatableModifier`
- `ControlActiveState`
- `ContentSizeCategory`

#### Gesture
- `RotationGesture`
- `MagnificationGesture`

#### View
- `ContextMenu`
- `NavigationView`

#### View Style
- `NavigationViewStyle`
- `StackNavigationViewStyle`
- `BorderlessButtonMenuStyle`
- `ColumnNavigationViewStyle`
- `DefaultNavigationViewStyle`
- `DoubleColumnNavigationViewStyle`