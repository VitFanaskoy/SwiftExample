# SwiftExample
//
//  ShapeEditorView.swift
//  CollageBuilder
//

import SwiftUI

struct ShapeEditorView: View {
    
    @EnvironmentObject private var store: AppStore
    
    @State private var showVideoTrim = false
    @State private var showMediaPicker = false
    @State private var previewImage: CIImage?
    
    var body: some View {
        Section("Common settings") {
            BlendModeSelectorView(blendMode: .init(
                get: { shape?.blendMode ?? .normal },
                set: { dispatch(.changeBlendMode($0)) }
            ))
            ZPositionSelectorView(zPosition: .init(
                get: { shape?.zPosition ?? 0 },
                set: { dispatch(.changeZPosition($0)) }
            ))
            addMedia
        }
        
        if let settings = shape?.media?.videoSettings {
            Section("video settings") {
                VStack {
                    videoTrim
                    CommonSliderView(
                        value: .init(
                            get: { CGFloat(settings.speed) },
                            set: { dispatchSettings(.changeSpeed($0)) }),
                        range: 0.2...3
                    )
                    Toggle("Muted", isOn: .init(
                        get: { settings.isMuted },
                        set: { dispatchSettings(.changeIsMute($0)) }
                    ))
                }
            }
        }
        
        if case .image(let image) = shape?.media?.resource {
            Section("background removal") {
                Toggle("remove", isOn: .init(
                    get: { shape?.media?.maskSettings != nil },
                    set: {
                        $0
                        ? dispatchMask(.createMask(originalImage: image))
                        : dispatchMask(.change(nil))
                    }
                ))
            }
        }
        
        Section("Blur") {
            if let shape {
                BlurSelectorView(blur: .init(
                    get: { shape.blur },
                    set: { dispatch(.changeBlur($0)) }
                ))
            }
        }
        
        Section("Adjustments") {
            if let shape {
                AdjustmentsSelectorView(adjustments: .init(
                    get: { shape.adjustments },
                    set: { dispatch(.changeAdjustments($0)) }
                ))
            }
        }
        
        if let previewImage {
            Section("Filters") {
                FiltersSelectorView(
                    filter: .init(
                        get: { shape?.filter },
                        set: { dispatch(.changeFilter($0)) }
                    ),
                    preview: previewImage
                )
            }
        }
        
        Section("manage") {
            remove
        }
        .onChange(of: shape?.id) { _ in
            Task { previewImage = await shape?.media?.image }
        }
        .task { previewImage = await shape?.media?.image }
    }
    
    private var videoTrim: some View {
        VStack {
            HStack {
                Text("trim")
                Spacer()
                Button {
                    showVideoTrim.toggle()
                } label: {
                    Image(systemName: "scissors")
                        .font(.title2)
                }
            }
        }
        .fullScreenCover(isPresented: $showVideoTrim) {
            if case .video(let video) = shape?.media?.resource,
               let id = store.state.selectedElement?.shapeId {
                VideoTrimView(trim: video.settings.trim,
                              url: video.videoUrl,
                              id: id)
                .presentationBackground(.regularMaterial)
            }
        }
    }
    
    private var remove: some View {
        HStack {
            Text("Remove")
            Spacer()
            Button {
                if let shape {
                    store.dispatch(.changeCollage(
                        .removeShape(shape.id)
                    ))
                }
            } label: {
                Image(systemName: "trash.slash")
                    .font(.title2)
            }
        }
    }
    
    private var addMedia: some View {
        HStack {
            Text("add media")
            Spacer()
            Button {
                showMediaPicker.toggle()
            } label: {
                Image(systemName: "plus.app")
                    .font(.largeTitle)
            }
        }
        .sheet(isPresented: $showMediaPicker) {
            MediaPickerView(media: .init(
                get: { nil } ,
                set: { dispatch(.changeMedia(.replace($0))) }
            ))
        }
    }
    
    private var shape: ShapeData? {
        let shape = store.state.collage.shapes.first(where: {
            $0.id == store.state.selectedElement?.shapeId
        })
        
        return shape
    }
    
    private func dispatchSettings(_ action: VideoSettingsModification) {
        dispatch(.changeMedia(.changeVideoSettings(action)))
    }
        
        private func dispatchMask(_ action: MaskSettingsModification) {
            dispatch(.changeMedia(.changeMask(action)))
        }
    
    private func dispatch(_ action: ShapeModification) {
        guard let id = store.state.selectedElement?.shapeId else {
            return
        }
        
        store.dispatch(.changeCollage(.changeShape(action, id: id)))
    }
}

struct ShapeEditorView_Previews: PreviewProvider {
    static var previews: some View {
        List {
            ShapeEditorView()
                .environmentObject(AppStore.preview)
        }
    }
}

