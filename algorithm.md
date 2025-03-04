# TagIdea 구현 알고리즘

이 문서는 TagIdea 앱의 주요 기능인 태그 입력(기능 1.1)과 알람 설정(기능 2.1)의 구현 알고리즘에 대해 설명합니다.

## 기능 1.1: 사용자가 직접 태그 입력

### 알고리즘 개요
1. 사용자가 앱에서 사진을 캡처하거나 갤러리에서 선택합니다.
2. 캡처 후 자동으로 태그 입력 팝업이 표시됩니다.
3. 사용자가 태그(쉼표로 구분)와 코멘트를 입력합니다.
4. 입력된 태그를 쉼표로 분리하여 배열로 저장합니다.
5. 사진과 태그를 메타데이터로 저장합니다.

### 구현 단계

#### 1. 사진 캡처 구현
```swift
import SwiftUI
import UIKit
import PhotosUI

struct CameraView: View {
    @State private var showImagePicker = false
    @State private var showTagPopup = false
    @State private var capturedImage: UIImage?
    
    var body: some View {
        VStack {
            // 카메라 버튼
            Button("사진 촬영") {
                showImagePicker = true
            }
            
            // 이미지 표시 영역
            if let image = capturedImage {
                Image(uiImage: image)
                    .resizable()
                    .scaledToFit()
            }
        }
        .sheet(isPresented: $showImagePicker) {
            ImagePicker(image: $capturedImage, showTagPopup: $showTagPopup)
        }
        .sheet(isPresented: $showTagPopup) {
            TagInputView(image: $capturedImage)
        }
    }
}

// 이미지 피커 구현
struct ImagePicker: UIViewControllerRepresentable {
    @Binding var image: UIImage?
    @Binding var showTagPopup: Bool
    
    func makeUIViewController(context: Context) -> UIImagePickerController {
        let picker = UIImagePickerController()
        picker.delegate = context.coordinator
        picker.sourceType = .camera
        return picker
    }
    
    func updateUIViewController(_ uiViewController: UIImagePickerController, context: Context) {}
    
    func makeCoordinator() -> Coordinator {
        Coordinator(self)
    }
    
    class Coordinator: NSObject, UIImagePickerControllerDelegate, UINavigationControllerDelegate {
        let parent: ImagePicker
        
        init(_ parent: ImagePicker) {
            self.parent = parent
        }
        
        func imagePickerController(_ picker: UIImagePickerController, 
                                  didFinishPickingMediaWithInfo info: [UIImagePickerController.InfoKey : Any]) {
            if let image = info[.originalImage] as? UIImage {
                parent.image = image
                
                // 이미지 선택 후 피커를 닫고 태그 입력 팝업 표시
                picker.dismiss(animated: true) {
                    self.parent.showTagPopup = true
                }
            }
        }
    }
}
```

#### 2. 태그 입력 팝업 구현
```swift
struct TagInputView: View {
    @Binding var image: UIImage?
    @State private var tags = ""
    @State private var comment = ""
    @Environment(\.presentationMode) var presentationMode
    @State private var showAlarmSetup = false
    
    var body: some View {
        NavigationView {
            Form {
                Section(header: Text("태그")) {
                    TextField("태그 입력 (쉼표로 구분)", text: $tags)
                }
                
                Section(header: Text("코멘트")) {
                    TextField("코멘트 입력", text: $comment)
                }
                
                Button("알람 설정") {
                    showAlarmSetup = true
                }
            }
            .navigationTitle("태그 입력")
            .navigationBarItems(
                leading: Button("취소") {
                    presentationMode.wrappedValue.dismiss()
                },
                trailing: Button("저장") {
                    savePhotoWithTags()
                    presentationMode.wrappedValue.dismiss()
                }
            )
            .sheet(isPresented: $showAlarmSetup) {
                AlarmSetupView(image: $image, tags: $tags, comment: $comment)
            }
        }
    }
    
    func savePhotoWithTags() {
        guard let image = image else { return }
        
        // 태그 파싱
        let tagArray = tags.split(separator: ",").map { 
            String($0.trimmingCharacters(in: .whitespaces)) 
        }
        
        // 사진 및 메타데이터 저장 로직
        saveImageWithMetadata(image: image, tags: tagArray, comment: comment)
    }
    
    func saveImageWithMetadata(image: UIImage, tags: [String], comment: String) {
        // 이미지를 데이터로 변환
        guard let imageData = image.jpegData(compressionQuality: 1.0) else { return }
        
        // PHPhotoLibrary에 저장
        PHPhotoLibrary.shared().performChanges {
            let creationRequest = PHAssetCreationRequest.forAsset()
            creationRequest.addResource(with: .photo, data: imageData, options: nil)
            
            // 메타데이터 설정 (IPTC/XMP 표준)
            if let metadata = createMetadata(tags: tags, comment: comment) {
                creationRequest.location = nil // 필요시 위치 정보 설정
                // 메타데이터 추가 로직 (실제 구현에서는 PhotoKit의 API 사용)
            }
        } completion: { success, error in
            if let error = error {
                print("Error saving photo: \(error.localizedDescription)")
            }
        }
    }
    
    func createMetadata(tags: [String], comment: String) -> [String: Any]? {
        // 실제 메타데이터 생성 로직 (IPTC/XMP 표준 준수)
        var metadata: [String: Any] = [:]
        
        // IPTC 태그 설정
        var iptc: [String: Any] = [:]
        iptc["Keywords"] = tags
        iptc["Caption/Abstract"] = comment
        
        metadata["IPTC"] = iptc
        
        return metadata
    }
}
```

## 기능 2.1: 사진 확인 알람 설정

### 알고리즘 개요
1. 태그 입력 후 알람 설정 옵션을 표시합니다.
2. 사용자가 날짜와 시간을 선택합니다.
3. 선택된 시간에 로컬 알림을 예약합니다.
4. 알림이 울리면 사진을 표시하거나 앱으로 이동합니다.

### 구현 단계

#### 1. 알람 설정 화면 구현
```swift
import SwiftUI
import UserNotifications

struct AlarmSetupView: View {
    @Binding var image: UIImage?
    @Binding var tags: String
    @Binding var comment: String
    @State private var alarmDate = Date().addingTimeInterval(3600) // 기본값 1시간 후
    @Environment(\.presentationMode) var presentationMode
    
    var body: some View {
        NavigationView {
            Form {
                Section(header: Text("알람 시간 설정")) {
                    DatePicker("알람 시간", 
                              selection: $alarmDate, 
                              displayedComponents: [.date, .hourAndMinute])
                }
                
                // 예정된 알람 확인용 텍스트
                Text("알람이 \(alarmDate.formatted(date: .long, time: .standard))에 설정됩니다.")
                    .font(.footnote)
                    .foregroundColor(.secondary)
            }
            .navigationTitle("알람 설정")
            .navigationBarItems(
                leading: Button("취소") {
                    presentationMode.wrappedValue.dismiss()
                },
                trailing: Button("설정") {
                    scheduleNotification()
                    presentationMode.wrappedValue.dismiss()
                }
            )
        }
    }
    
    func scheduleNotification() {
        // 알림 권한 요청
        UNUserNotificationCenter.current().requestAuthorization(options: [.alert, .sound, .badge]) { granted, error in
            if granted {
                createNotification()
            }
        }
    }
    
    func createNotification() {
        // 알림 내용 설정
        let content = UNMutableNotificationContent()
        content.title = "사진 확인 알림"
        content.body = "태그: \(tags)"
        content.sound = .default
        
        // 이미지 첨부 (기기에 임시 파일로 저장)
        if let image = image, let imageData = image.jpegData(compressionQuality: 0.7) {
            let tempDir = FileManager.default.temporaryDirectory
            let imageURL = tempDir.appendingPathComponent("alarm_image.jpg")
            
            do {
                try imageData.write(to: imageURL)
                
                if let attachment = try? UNNotificationAttachment(identifier: "imageAttachment", 
                                                                url: imageURL, 
                                                                options: nil) {
                    content.attachments = [attachment]
                }
            } catch {
                print("Error saving notification image: \(error)")
            }
        }
        
        // 알림 트리거 설정 (지정된 시간)
        let triggerDate = Calendar.current.dateComponents([.year, .month, .day, .hour, .minute], 
                                                         from: alarmDate)
        let trigger = UNCalendarNotificationTrigger(dateMatching: triggerDate, repeats: false)
        
        // 알림 요청 생성
        let notificationID = UUID().uuidString
        let request = UNNotificationRequest(identifier: notificationID, 
                                          content: content, 
                                          trigger: trigger)
        
        // 알림 스케줄링
        UNUserNotificationCenter.current().add(request) { error in
            if let error = error {
                print("Error scheduling notification: \(error)")
            } else {
                // 알람 정보 로컬 DB에 저장 (Core Data 사용)
                saveAlarmToDB(id: notificationID, date: alarmDate, tags: tags, comment: comment)
            }
        }
    }
    
    func saveAlarmToDB(id: String, date: Date, tags: String, comment: String) {
        // Core Data를 사용하여 알람 정보 저장
        // 실제 구현에서는 Core Data 모델 및 컨텍스트를 활용
    }
}
```

#### 2. 앱 시작 시 알림 권한 요청 및 처리
```swift
@main
struct TagCaptureApp: App {
    @UIApplicationDelegateAdaptor(AppDelegate.self) var appDelegate
    
    var body: some Scene {
        WindowGroup {
            ContentView()
        }
    }
}

class AppDelegate: NSObject, UIApplicationDelegate, UNUserNotificationCenterDelegate {
    func application(_ application: UIApplication, didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey : Any]? = nil) -> Bool {
        // 알림 권한 요청
        UNUserNotificationCenter.current().requestAuthorization(options: [.alert, .sound, .badge]) { granted, error in
            print("Notification permission granted: \(granted)")
        }
        
        // 알림 델리게이트 설정
        UNUserNotificationCenter.current().delegate = self
        
        return true
    }
    
    // 앱이 포그라운드 상태일 때도 알림 표시
    func userNotificationCenter(_ center: UNUserNotificationCenter, 
                              willPresent notification: UNNotification, 
                              withCompletionHandler completionHandler: @escaping (UNNotificationPresentationOptions) -> Void) {
        completionHandler([.banner, .sound])
    }
    
    // 알림을 탭했을 때 처리
    func userNotificationCenter(_ center: UNUserNotificationCenter, 
                              didReceive response: UNNotificationResponse, 
                              withCompletionHandler completionHandler: @escaping () -> Void) {
        // 알림 ID를 통해 관련 사진 정보 로딩
        let notificationID = response.notification.request.identifier
        
        // 여기서 해당 알림 ID에 연결된 사진 표시
        // NotificationCenter를 통해 앱 내 화면 전환 알림 등을 처리
        
        completionHandler()
    }
}
```

#### 3. 알람 관리 화면 구현
```swift
struct AlarmListView: View {
    @State private var alarms: [AlarmItem] = [] // 알람 모델 배열
    
    var body: some View {
        NavigationView {
            List {
                ForEach(alarms) { alarm in
                    AlarmRow(alarm: alarm)
                }
                .onDelete(perform: deleteAlarm)
            }
            .navigationTitle("알람 목록")
            .onAppear {
                loadAlarms()
            }
        }
    }
    
    func loadAlarms() {
        // Core Data에서 알람 정보 로드
        // 실제 구현에서는 Core Data 쿼리 사용
    }
    
    func deleteAlarm(at offsets: IndexSet) {
        for index in offsets {
            let alarm = alarms[index]
            
            // 예약된 알림 취소
            UNUserNotificationCenter.current().removePendingNotificationRequests(
                withIdentifiers: [alarm.id]
            )
            
            // Core Data에서 알람 삭제
            // 실제 구현에서는 Core Data 삭제 로직 사용
        }
        
        // UI 목록에서 삭제
        alarms.remove(atOffsets: offsets)
    }
}

// 알람 항목 UI 컴포넌트
struct AlarmRow: View {
    let alarm: AlarmItem
    
    var body: some View {
        HStack {
            VStack(alignment: .leading) {
                Text(alarm.tags)
                    .font(.headline)
                Text(alarm.date.formatted(date: .long, time: .shortened))
                    .font(.subheadline)
            }
            
            Spacer()
            
            // 알람 미리보기 이미지 (옵션)
            if let imageData = alarm.imageData, let uiImage = UIImage(data: imageData) {
                Image(uiImage: uiImage)
                    .resizable()
                    .scaledToFill()
                    .frame(width: 60, height: 60)
                    .clipShape(RoundedRectangle(cornerRadius: 8))
            }
        }
        .padding(.vertical, 8)
    }
}

// 알람 데이터 모델
struct AlarmItem: Identifiable {
    let id: String
    let date: Date
    let tags: String
    let comment: String
    let imageData: Data?
    let status: AlarmStatus
    
    enum AlarmStatus: String {
        case pending
        case completed
        case cancelled
    }
}
```

## 데이터 흐름

### 태그 및 알람 저장 흐름
1. 사용자가 사진을 촬영하거나 갤러리에서 선택
2. 태그 입력 팝업 표시
3. 사용자가 태그와 코멘트 입력
4. 알람 설정 옵션 선택 (선택적)
5. 사진, 태그, 코멘트 저장
   - 사진은 Photos 라이브러리에 저장
   - 태그와 코멘트는 메타데이터로 저장
   - 알람 정보는 Core Data에 저장
6. 알람 설정 시 UserNotifications 프레임워크로 로컬 알림 예약

### 태그 검색 흐름
1. 사용자가 검색 탭에서 키워드 입력
2. Core Data 또는 PhotoKit을 통해 태그와 메타데이터 검색
3. 검색 결과 표시

### 알람 처리 흐름
1. 설정된 시간에 로컬 알림 발생
2. 사용자가 알림 탭
3. 앱이 해당 알림 ID에 연결된 사진 정보 로드
4. 해당 사진과 태그 정보 표시 
