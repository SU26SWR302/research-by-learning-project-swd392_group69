# Tài liệu ERD và triển khai JPA cho module lập lịch chiếu OptiCine

## 1. Phạm vi module

Tài liệu này mô tả thiết kế ERD và hướng triển khai JPA cho nhóm chức năng quản trị/lập lịch chiếu của OptiCine.

Các chức năng chính:

- Movie Management: quản lý phim, trạng thái phim, thời lượng, độ ưu tiên hoặc độ phổ biến.
- Room Management: quản lý phòng chiếu, sức chứa, loại phòng, trạng thái phòng.
- Showtime Management: tạo và quản lý suất chiếu.
- Auto Scheduling: tự động đề xuất lịch chiếu dựa trên phim, phòng, khung giờ và doanh thu dự kiến.
- Approval Workflow: quy trình gửi, duyệt, trả về hoặc từ chối lịch chiếu.
- Revenue Estimation: ước tính doanh thu/sức chứa dự kiến theo suất chiếu.
- User/Role Management: quản lý người dùng, vai trò và quyền.

## 2. ERD dạng DBML

Có thể dán đoạn DBML này vào dbdiagram.io để hiển thị ERD.

```dbml
Table users {
  id bigint [pk, increment]
  full_name varchar(255) [not null]
  email varchar(255) [not null, unique]
  password varchar(255) [not null]
  status varchar(50) [not null] // ACTIVE, INACTIVE
  created_at datetime
  updated_at datetime
}

Table roles {
  id bigint [pk, increment]
  name varchar(100) [not null, unique] // STAFF, SHIFT_SUPERVISOR, BRANCH_MANAGER, ADMIN
  description varchar(255)
}

Table permissions {
  id bigint [pk, increment]
  name varchar(100) [not null, unique]
  description varchar(255)
}

Table user_roles {
  user_id bigint [not null]
  role_id bigint [not null]

  indexes {
    (user_id, role_id) [pk]
  }
}

Table role_permissions {
  role_id bigint [not null]
  permission_id bigint [not null]

  indexes {
    (role_id, permission_id) [pk]
  }
}

Table movies {
  id bigint [pk, increment]
  title varchar(255) [not null]
  description text
  duration_minutes int [not null]
  genre varchar(100)
  language varchar(100)
  age_rating varchar(50)
  release_date date
  poster_url varchar(500)
  trailer_url varchar(500)
  popularity_score double [not null]
  status varchar(50) [not null] // CREATED, ACTIVE, DISABLED, ARCHIVED
  created_at datetime
  updated_at datetime
}

Table rooms {
  id bigint [pk, increment]
  room_code varchar(50) [not null, unique]
  name varchar(100) [not null]
  capacity int [not null]
  room_type varchar(50) // STANDARD, VIP, IMAX
  status varchar(50) [not null] // AVAILABLE, BUSY, MAINTENANCE, DISABLED
  created_at datetime
  updated_at datetime
}

Table room_maintenances {
  id bigint [pk, increment]
  room_id bigint [not null]
  reason varchar(255)
  start_time datetime [not null]
  end_time datetime [not null]
  status varchar(50) [not null] // SCHEDULED, IN_PROGRESS, DONE, CANCELLED
  created_by bigint
  created_at datetime
}

Table time_slots {
  id bigint [pk, increment]
  name varchar(100)
  start_time time [not null]
  end_time time [not null]
  weight double [not null]
  slot_type varchar(50) // NORMAL, PRIME_TIME
}

Table showtimes {
  id bigint [pk, increment]
  movie_id bigint [not null]
  room_id bigint [not null]
  schedule_id bigint
  start_time datetime [not null]
  end_time datetime [not null]
  cleaning_buffer_minutes int [not null]
  estimated_occupancy double
  estimated_revenue double
  status varchar(50) [not null] // DRAFT, APPROVED, PUBLISHED, CANCELLED
  created_at datetime
  updated_at datetime

  indexes {
    (room_id, start_time, end_time) [name: "idx_showtime_room_time"]
    (movie_id)
    (schedule_id)
  }
}

Table schedules {
  id bigint [pk, increment]
  schedule_date date [not null]
  schedule_type varchar(50) [not null] // DAILY, WEEKLY
  strategy_type varchar(100) [not null] // REVENUE_MAXIMIZE, DIVERSITY
  status varchar(50) [not null] // DRAFT, SUBMITTED, SUPERVISOR_APPROVED, MANAGER_APPROVED, PUBLISHED, REJECTED, ARCHIVED
  total_estimated_revenue double
  created_by bigint
  created_at datetime
  updated_at datetime
}

Table schedule_versions {
  id bigint [pk, increment]
  schedule_id bigint [not null]
  version_number int [not null]
  snapshot_data text
  change_note varchar(500)
  created_by bigint
  created_at datetime
}

Table approval_requests {
  id bigint [pk, increment]
  schedule_id bigint [not null]
  current_step varchar(100) [not null] // SHIFT_SUPERVISOR, BRANCH_MANAGER
  status varchar(50) [not null] // PENDING, APPROVED, REJECTED, RETURNED, CANCELLED
  requested_by bigint [not null]
  assigned_to bigint
  created_at datetime
  updated_at datetime
}

Table approval_comments {
  id bigint [pk, increment]
  approval_request_id bigint [not null]
  user_id bigint [not null]
  comment text
  action varchar(50) [not null] // SUBMIT, APPROVE, REJECT, RETURN, PUBLISH
  created_at datetime
}

Table system_configurations {
  id bigint [pk, increment]
  config_key varchar(100) [not null, unique]
  config_value varchar(500) [not null]
  description varchar(255)
  updated_by bigint
  updated_at datetime
}

Table reports {
  id bigint [pk, increment]
  report_name varchar(255) [not null]
  report_type varchar(100) [not null] // REVENUE, OCCUPANCY, ROOM_UTILIZATION
  from_date date
  to_date date
  generated_file_url varchar(500)
  generated_by bigint
  created_at datetime
}

Table notifications {
  id bigint [pk, increment]
  user_id bigint [not null]
  title varchar(255) [not null]
  message text
  type varchar(100) // APPROVAL, ALERT, SYSTEM
  is_read boolean [not null]
  created_at datetime
}

Table audit_logs {
  id bigint [pk, increment]
  user_id bigint
  action varchar(255) [not null]
  entity_name varchar(100)
  entity_id bigint
  old_value text
  new_value text
  ip_address varchar(100)
  created_at datetime
}

Ref: users.id < user_roles.user_id
Ref: roles.id < user_roles.role_id

Ref: roles.id < role_permissions.role_id
Ref: permissions.id < role_permissions.permission_id

Ref: rooms.id < room_maintenances.room_id
Ref: users.id < room_maintenances.created_by

Ref: movies.id < showtimes.movie_id
Ref: rooms.id < showtimes.room_id
Ref: schedules.id < showtimes.schedule_id

Ref: users.id < schedules.created_by

Ref: schedules.id < schedule_versions.schedule_id
Ref: users.id < schedule_versions.created_by

Ref: schedules.id < approval_requests.schedule_id
Ref: users.id < approval_requests.requested_by
Ref: users.id < approval_requests.assigned_to

Ref: approval_requests.id < approval_comments.approval_request_id
Ref: users.id < approval_comments.user_id

Ref: users.id < system_configurations.updated_by
Ref: users.id < reports.generated_by
Ref: users.id < notifications.user_id
Ref: users.id < audit_logs.user_id
```

## 3. Nhóm bảng chính

| Nhóm | Bảng | Mục đích |
| --- | --- | --- |
| Người dùng và phân quyền | `users`, `roles`, `permissions`, `user_roles`, `role_permissions` | Quản lý tài khoản, vai trò và quyền thao tác trong hệ thống. |
| Phim và phòng chiếu | `movies`, `rooms`, `room_maintenances`, `time_slots` | Lưu thông tin phim, phòng chiếu, trạng thái bảo trì và khung giờ chiếu. |
| Lập lịch | `showtimes`, `schedules`, `schedule_versions` | Quản lý lịch chiếu, suất chiếu và lịch sử phiên bản lịch. |
| Phê duyệt | `approval_requests`, `approval_comments` | Theo dõi luồng gửi duyệt, duyệt, trả về, từ chối và ghi chú phê duyệt. |
| Vận hành | `system_configurations`, `reports`, `notifications`, `audit_logs` | Cấu hình hệ thống, báo cáo, thông báo và nhật ký thao tác. |

## 4. Phân tích quan hệ chính và mapping JPA

### 4.1 Movie 1 - N Showtime

Một phim có thể được chiếu nhiều lần trong ngày hoặc nhiều ngày khác nhau. Ví dụ phim Conan có thể có các suất chiếu ở Room A lúc 09:00, Room B lúc 13:00 và Room A lúc 20:00.

Trong JPA:

- `Movie`: `@OneToMany(mappedBy = "movie")`
- `Showtime`: `@ManyToOne`

```java
@Entity
@Table(name = "movies")
public class Movie {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String title;
    private Integer durationMinutes;
    private Double popularityScore;

    @Enumerated(EnumType.STRING)
    private MovieStatus status;

    @OneToMany(mappedBy = "movie")
    private List<Showtime> showtimes = new ArrayList<>();
}
```

```java
@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "movie_id", nullable = false)
private Movie movie;
```

### 4.2 Room 1 - N Showtime

Một phòng chiếu có nhiều suất chiếu. Tuy nhiên, tại cùng một thời điểm, một phòng không được có hai suất chiếu bị trùng thời gian.

Trong JPA:

- `Room`: `@OneToMany(mappedBy = "room")`
- `Showtime`: `@ManyToOne`

```java
@Entity
@Table(name = "rooms")
public class Room {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String roomCode;
    private String name;
    private Integer capacity;
    private String roomType;

    @Enumerated(EnumType.STRING)
    private RoomStatus status;

    @OneToMany(mappedBy = "room")
    private List<Showtime> showtimes = new ArrayList<>();
}
```

```java
@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "room_id", nullable = false)
private Room room;
```

### 4.3 Schedule 1 - N Showtime

Một lịch chiếu chứa nhiều suất chiếu. Trong ERD, bảng `schedules` đại diện cho lịch ngày hoặc tuần. Trong code hiện tại của dự án có entity `SchedulePlan`; nếu tiếp tục dùng tên này thì `SchedulePlan` có thể được xem là entity tương ứng với bảng `schedules`.

Ví dụ lịch ngày `2026-06-01` gồm:

- Showtime 1: Conan - Room A - 09:00
- Showtime 2: Doraemon - Room B - 10:00
- Showtime 3: Avengers - Room C - 20:00

Trong JPA:

- `Schedule`: `@OneToMany(mappedBy = "schedule")`
- `Showtime`: `@ManyToOne`

```java
@Entity
@Table(name = "schedules")
public class Schedule {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private LocalDate scheduleDate;
    private Double totalEstimatedRevenue;

    @Enumerated(EnumType.STRING)
    private ScheduleStatus status;

    @OneToMany(mappedBy = "schedule", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<Showtime> showtimes = new ArrayList<>();
}
```

```java
@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "schedule_id")
private Schedule schedule;
```

### 4.4 Schedule 1 - N ApprovalRequest

Một lịch chiếu có thể trải qua nhiều bước phê duyệt, ví dụ: nhân viên gửi lịch, ca trưởng duyệt, quản lý chi nhánh duyệt.

Trong JPA:

- `Schedule`: `@OneToMany(mappedBy = "schedule")`
- `ApprovalRequest`: `@ManyToOne`

```java
@OneToMany(mappedBy = "schedule", cascade = CascadeType.ALL, orphanRemoval = true)
private List<ApprovalRequest> approvalRequests = new ArrayList<>();
```

```java
@Entity
@Table(name = "approval_requests")
public class ApprovalRequest {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String currentStep;

    @Enumerated(EnumType.STRING)
    private ApprovalStatus status;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "schedule_id", nullable = false)
    private Schedule schedule;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "requested_by", nullable = false)
    private User requestedBy;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "assigned_to")
    private User assignedTo;
}
```

### 4.5 ApprovalRequest 1 - N ApprovalComment

Một yêu cầu phê duyệt có thể có nhiều comment/action để lưu lịch sử xử lý.

Trong JPA:

- `ApprovalRequest`: `@OneToMany(mappedBy = "approvalRequest")`
- `ApprovalComment`: `@ManyToOne`

```java
@OneToMany(mappedBy = "approvalRequest", cascade = CascadeType.ALL, orphanRemoval = true)
private List<ApprovalComment> comments = new ArrayList<>();
```

```java
@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "approval_request_id", nullable = false)
private ApprovalRequest approvalRequest;

@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "user_id", nullable = false)
private User user;
```

### 4.6 User 1 - N ApprovalRequest

Một user có thể tạo nhiều yêu cầu phê duyệt và cũng có thể là người được giao xử lý nhiều yêu cầu phê duyệt.

Trong JPA:

- `User`: `@OneToMany(mappedBy = "requestedBy")`
- `User`: `@OneToMany(mappedBy = "assignedTo")`
- `ApprovalRequest`: `@ManyToOne`

```java
@OneToMany(mappedBy = "requestedBy")
private List<ApprovalRequest> requestedApprovals = new ArrayList<>();

@OneToMany(mappedBy = "assignedTo")
private List<ApprovalRequest> assignedApprovals = new ArrayList<>();
```

## 5. Entity Showtime đề xuất

`Showtime` là entity trung tâm của module lập lịch, liên kết phim, phòng chiếu và lịch chiếu.

```java
@Entity
@Table(
    name = "showtimes",
    indexes = {
        @Index(name = "idx_showtime_room_time", columnList = "room_id,start_time,end_time")
    }
)
public class Showtime {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "movie_id", nullable = false)
    private Movie movie;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "room_id", nullable = false)
    private Room room;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "schedule_id")
    private Schedule schedule;

    @Column(name = "start_time", nullable = false)
    private LocalDateTime startTime;

    @Column(name = "end_time", nullable = false)
    private LocalDateTime endTime;

    @Column(name = "cleaning_buffer_minutes", nullable = false)
    private Integer cleaningBufferMinutes;

    @Column(name = "estimated_occupancy")
    private Double estimatedOccupancy;

    @Column(name = "estimated_revenue")
    private Double estimatedRevenue;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 50)
    private ShowtimeStatus status;
}
```

## 6. Enum đề xuất

Nên dùng enum thay vì lưu trạng thái bằng chuỗi tự do để giảm lỗi nhập sai trạng thái.

```java
public enum MovieStatus {
    CREATED,
    ACTIVE,
    DISABLED,
    ARCHIVED
}

public enum RoomStatus {
    AVAILABLE,
    BUSY,
    MAINTENANCE,
    DISABLED
}

public enum ShowtimeStatus {
    DRAFT,
    APPROVED,
    PUBLISHED,
    CANCELLED
}

public enum ScheduleStatus {
    DRAFT,
    SUBMITTED,
    SUPERVISOR_APPROVED,
    MANAGER_APPROVED,
    PUBLISHED,
    REJECTED,
    ARCHIVED
}

public enum ApprovalStatus {
    PENDING,
    APPROVED,
    REJECTED,
    RETURNED,
    CANCELLED
}
```

## 7. Logic kiểm tra trùng lịch chiếu

Một suất chiếu mới bị xem là trùng lịch nếu khoảng thời gian mới giao với khoảng thời gian đã tồn tại trong cùng một phòng:

```text
newStart < existingEnd
AND
newEnd > existingStart
```

Repository:

```java
public interface ShowtimeRepository extends JpaRepository<Showtime, Long> {

    @Query("""
        SELECT COUNT(s) > 0
        FROM Showtime s
        WHERE s.room.id = :roomId
          AND s.startTime < :newEnd
          AND s.endTime > :newStart
          AND s.status <> com.opticine.booking.entity.ShowtimeStatus.CANCELLED
    """)
    boolean existsConflict(
        @Param("roomId") Long roomId,
        @Param("newStart") LocalDateTime newStart,
        @Param("newEnd") LocalDateTime newEnd
    );
}
```

Nếu có thời gian dọn phòng, `endTime` nên bao gồm cả buffer:

```java
LocalDateTime endTime = startTime
    .plusMinutes(movie.getDurationMinutes())
    .plusMinutes(cleaningBufferMinutes);
```

## 8. Service tạo Showtime

Service cần kiểm tra phim, phòng, trạng thái hợp lệ, thời gian kết thúc, xung đột lịch và doanh thu dự kiến trước khi lưu.

```java
@Service
public class ShowtimeService {

    private final MovieRepository movieRepository;
    private final RoomRepository roomRepository;
    private final ShowtimeRepository showtimeRepository;

    public ShowtimeService(
            MovieRepository movieRepository,
            RoomRepository roomRepository,
            ShowtimeRepository showtimeRepository
    ) {
        this.movieRepository = movieRepository;
        this.roomRepository = roomRepository;
        this.showtimeRepository = showtimeRepository;
    }

    @Transactional
    public Showtime createShowtime(Long movieId, Long roomId, LocalDateTime startTime) {
        Movie movie = movieRepository.findById(movieId)
                .orElseThrow(() -> new RuntimeException("Movie not found"));

        Room room = roomRepository.findById(roomId)
                .orElseThrow(() -> new RuntimeException("Room not found"));

        if (movie.getStatus() != MovieStatus.ACTIVE) {
            throw new RuntimeException("Movie is not active");
        }

        if (room.getStatus() != RoomStatus.AVAILABLE) {
            throw new RuntimeException("Room is not available");
        }

        int cleaningBuffer = 15;
        LocalDateTime endTime = startTime
                .plusMinutes(movie.getDurationMinutes())
                .plusMinutes(cleaningBuffer);

        boolean conflict = showtimeRepository.existsConflict(roomId, startTime, endTime);
        if (conflict) {
            throw new RuntimeException("Showtime conflict detected");
        }

        double estimatedRevenue = movie.getPopularityScore() * room.getCapacity();

        Showtime showtime = new Showtime();
        showtime.setMovie(movie);
        showtime.setRoom(room);
        showtime.setStartTime(startTime);
        showtime.setEndTime(endTime);
        showtime.setCleaningBufferMinutes(cleaningBuffer);
        showtime.setEstimatedRevenue(estimatedRevenue);
        showtime.setStatus(ShowtimeStatus.DRAFT);

        return showtimeRepository.save(showtime);
    }
}
```

## 9. Luồng nghiệp vụ đề xuất

1. Staff hoặc hệ thống tự động tạo `Schedule` ở trạng thái `DRAFT`.
2. Thêm nhiều `Showtime` vào `Schedule`.
3. Khi thêm showtime, hệ thống kiểm tra phòng, phim và trùng lịch.
4. Tính `estimatedRevenue` và `estimatedOccupancy` cho từng showtime.
5. Tổng hợp `totalEstimatedRevenue` cho schedule.
6. Staff submit schedule, hệ thống tạo `ApprovalRequest`.
7. Shift Supervisor duyệt hoặc trả về.
8. Branch Manager duyệt cuối cùng.
9. Khi được duyệt, schedule chuyển sang `PUBLISHED` và các showtime có thể chuyển sang `PUBLISHED`.

## 10. Tóm tắt mapping quan hệ JPA

| Quan hệ | Mapping phía cha | Mapping phía con |
| --- | --- | --- |
| Movie 1 - N Showtime | `Movie.@OneToMany(mappedBy = "movie")` | `Showtime.@ManyToOne movie` |
| Room 1 - N Showtime | `Room.@OneToMany(mappedBy = "room")` | `Showtime.@ManyToOne room` |
| Schedule 1 - N Showtime | `Schedule.@OneToMany(mappedBy = "schedule")` | `Showtime.@ManyToOne schedule` |
| Schedule 1 - N ApprovalRequest | `Schedule.@OneToMany(mappedBy = "schedule")` | `ApprovalRequest.@ManyToOne schedule` |
| ApprovalRequest 1 - N ApprovalComment | `ApprovalRequest.@OneToMany(mappedBy = "approvalRequest")` | `ApprovalComment.@ManyToOne approvalRequest` |
| User 1 - N ApprovalRequest | `User.@OneToMany(mappedBy = "requestedBy/assignedTo")` | `ApprovalRequest.@ManyToOne requestedBy/assignedTo` |

## 11. Ghi chú khi áp dụng vào code hiện tại

- Code hiện tại đã có các entity nền như `Movie`, `Room`, `Showtime`, `TimeSlot`, `SchedulePlan`, `User`, `Role`.
- Nếu nhóm muốn giữ tên `SchedulePlan`, có thể dùng `SchedulePlan` thay cho `Schedule`, nhưng nên thống nhất tên cột foreign key trong `showtimes`, ví dụ `schedule_plan_id` hoặc `schedule_id`.
- Entity hiện tại đang lưu nhiều trạng thái bằng `String`. Khi hoàn thiện module, nên đổi sang `@Enumerated(EnumType.STRING)` để thống nhất trạng thái và giảm lỗi dữ liệu.
- Với quan hệ `@ManyToOne`, nên dùng `fetch = FetchType.LAZY` để tránh load dữ liệu không cần thiết.
- Với collection `@OneToMany`, chỉ dùng `cascade = CascadeType.ALL` khi vòng đời entity con phụ thuộc vào entity cha. Không nên cascade tùy tiện với `Movie` hoặc `Room` sang `Showtime` vì xóa phim/phòng có thể ảnh hưởng nhiều dữ liệu lịch sử.
- Cần có index trên `(room_id, start_time, end_time)` để tối ưu kiểm tra trùng lịch.
- Logic kiểm tra trùng lịch phải bỏ qua showtime đã `CANCELLED`.

## 12. Kết luận

Bộ quan hệ phù hợp nhất cho MVP OptiCine là lấy `Showtime` làm trung tâm, liên kết với `Movie`, `Room` và `Schedule`. Phần phê duyệt được tách riêng qua `ApprovalRequest` và `ApprovalComment` để lưu được lịch sử xử lý. Thiết kế này đủ cho các chức năng quản lý phim, phòng, suất chiếu, tự động lập lịch, ước tính doanh thu và phê duyệt lịch chiếu.
