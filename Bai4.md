# Bài 4: Chiến Lược Sinh Dữ Liệu Giả Lập & Test Case

---

## I. Mega-Prompt Duy Nhất

```
Vai trò: Đóng vai một Senior QA Engineer kiêm Java Developer có kinh nghiệm viết Unit Test với JUnit 5, đang làm việc trong dự án E-commerce TechShop.

Mục tiêu: Thực hiện chuỗi tác vụ sau trong MỘT lần trả lời duy nhất:
  Tác vụ 1 — Sinh bộ dữ liệu giả lập (Mock Data) dạng JSON gồm chính xác 5 user, mỗi user đại diện cho 1 kịch bản kiểm thử cụ thể.
  Tác vụ 2 — Tự giả định logic hàm login(String username, String password) hợp lý dựa trên 5 kịch bản test, rồi viết mã nguồn class UserAuthentication hoàn chỉnh.
  Tác vụ 3 — Sinh mã nguồn JUnit 5 Test Class đầy đủ để test phương thức login() với 5 kịch bản từ bộ Mock Data.

Ngữ cảnh: 
  - Dự án TechShop đang phát triển module đăng nhập người dùng.
  - Class UserAuthentication có phương thức login(String username, String password) trả về boolean hoặc ném exception tùy kịch bản.
  - Hệ thống chưa có database test, cần dùng Mock Data trực tiếp trong code test.
  - Code cần đảm bảo chất lượng trước khi merge lên nhánh chính (main branch).

Ràng buộc:
  1. Bộ Mock Data JSON phải gồm đúng 5 user đại diện cho 5 kịch bản sau:
     - Kịch bản 1 (Happy Path): Thông tin đăng nhập đúng (username và password hợp lệ, tài khoản đang hoạt động).
     - Kịch bản 2 (Wrong Password): Username đúng nhưng password sai.
     - Kịch bản 3 (Locked Account): Tài khoản bị khóa (locked = true), dù username và password đúng.
     - Kịch bản 4 (SQL Injection): Username chứa chuỗi SQL Injection (ví dụ: ' OR '1'='1), password bất kỳ.
     - Kịch bản 5 (Empty Password): Username hợp lệ nhưng password là chuỗi rỗng ("").
  2. Mỗi user trong JSON phải có các trường: username, password, isLocked, expectedResult, description.
  3. Logic hàm login() cần tự giả định hợp lý:
     - Kiểm tra input không rỗng/null.
     - Phát hiện và chặn ký tự SQL Injection trong username.
     - Kiểm tra tài khoản bị khóa → ném AccountLockedException.
     - So khớp username/password với dữ liệu lưu trữ.
  4. Test class sử dụng JUnit 5 (org.junit.jupiter.api).
  5. Mỗi test method phải có @DisplayName mô tả rõ kịch bản bằng tiếng Việt.
  6. Sử dụng Assertions phù hợp: assertTrue, assertFalse, assertThrows.

Định dạng: Trả về kết quả gồm 3 phần rõ ràng:
  Phần 1 — Bộ Mock Data (JSON code block).
  Phần 2 — Class UserAuthentication (Java code block).
  Phần 3 — Class UserAuthenticationTest (Java code block).
```

---

## II. Bộ Mock Data (JSON)

```json
[
  {
    "username": "nguyen.vana",
    "password": "SecurePass@123",
    "isLocked": false,
    "expectedResult": "LOGIN_SUCCESS",
    "description": "Kịch bản 1 - Happy Path: Thông tin đăng nhập hoàn toàn hợp lệ, tài khoản đang hoạt động"
  },
  {
    "username": "tran.vanb",
    "password": "WrongPassword!456",
    "isLocked": false,
    "expectedResult": "LOGIN_FAILED",
    "description": "Kịch bản 2 - Wrong Password: Username tồn tại nhưng mật khẩu không khớp"
  },
  {
    "username": "le.vanc",
    "password": "CorrectPass@789",
    "isLocked": true,
    "expectedResult": "ACCOUNT_LOCKED",
    "description": "Kịch bản 3 - Locked Account: Tài khoản bị khóa do vi phạm chính sách, dù thông tin đăng nhập đúng"
  },
  {
    "username": "' OR '1'='1",
    "password": "anything",
    "isLocked": false,
    "expectedResult": "INVALID_INPUT",
    "description": "Kịch bản 4 - SQL Injection: Username chứa chuỗi tấn công SQL Injection"
  },
  {
    "username": "pham.vand",
    "password": "",
    "isLocked": false,
    "expectedResult": "INVALID_INPUT",
    "description": "Kịch bản 5 - Empty Password: Username hợp lệ nhưng mật khẩu là chuỗi rỗng"
  }
]
```

---

## III. Class UserAuthentication (Java)

```java
import java.util.HashMap;
import java.util.Map;
import java.util.regex.Pattern;

/**
 * Service xác thực đăng nhập người dùng cho hệ thống TechShop.
 *
 * <p>Thực hiện các bước kiểm tra theo thứ tự:</p>
 * <ol>
 *   <li>Validate input: kiểm tra username/password không rỗng hoặc null.</li>
 *   <li>Phát hiện SQL Injection: chặn các ký tự nguy hiểm trong username.</li>
 *   <li>Kiểm tra tài khoản tồn tại trong hệ thống.</li>
 *   <li>Kiểm tra trạng thái khóa tài khoản.</li>
 *   <li>So khớp mật khẩu.</li>
 * </ol>
 */
public class UserAuthentication {

    /** Pattern phát hiện các ký tự đặc biệt thường dùng trong SQL Injection */
    private static final Pattern SQL_INJECTION_PATTERN =
            Pattern.compile(".*['\";\\-\\-\\/\\*].*");

    /** Cơ sở dữ liệu giả lập lưu trữ thông tin người dùng (thay thế cho DB thực tế) */
    private final Map<String, UserRecord> userDatabase = new HashMap<>();

    /**
     * Record lưu trữ thông tin người dùng.
     */
    public record UserRecord(String username, String password, boolean isLocked) {}

    /**
     * Custom exception khi tài khoản bị khóa.
     */
    public static class AccountLockedException extends RuntimeException {
        public AccountLockedException(String message) {
            super(message);
        }
    }

    /**
     * Khởi tạo service với dữ liệu người dùng mặc định (Mock Data).
     */
    public UserAuthentication() {
        // Đăng ký các user giả lập vào "database"
        userDatabase.put("nguyen.vana",
                new UserRecord("nguyen.vana", "SecurePass@123", false));
        userDatabase.put("tran.vanb",
                new UserRecord("tran.vanb", "RealPassword@456", false));
        userDatabase.put("le.vanc",
                new UserRecord("le.vanc", "CorrectPass@789", true));
        userDatabase.put("pham.vand",
                new UserRecord("pham.vand", "ValidPass@000", false));
    }

    /**
     * Xác thực đăng nhập người dùng.
     *
     * @param username Tên đăng nhập. Không được null, rỗng hoặc chứa ký tự SQL Injection.
     * @param password Mật khẩu. Không được null hoặc rỗng.
     * @return {@code true} nếu đăng nhập thành công, {@code false} nếu sai thông tin.
     * @throws IllegalArgumentException nếu username/password rỗng hoặc chứa SQL Injection.
     * @throws AccountLockedException nếu tài khoản đang bị khóa.
     */
    public boolean login(String username, String password) {
        // Bước 1: Validate input — không chấp nhận null hoặc chuỗi rỗng
        if (username == null || username.isBlank()) {
            throw new IllegalArgumentException("Username không được để trống.");
        }
        if (password == null || password.isEmpty()) {
            throw new IllegalArgumentException("Password không được để trống.");
        }

        // Bước 2: Phát hiện SQL Injection trong username
        if (SQL_INJECTION_PATTERN.matcher(username).matches()) {
            throw new IllegalArgumentException(
                    "Username chứa ký tự không hợp lệ (nghi ngờ SQL Injection).");
        }

        // Bước 3: Kiểm tra user có tồn tại trong hệ thống không
        UserRecord user = userDatabase.get(username);
        if (user == null) {
            return false; // User không tồn tại
        }

        // Bước 4: Kiểm tra tài khoản bị khóa
        if (user.isLocked()) {
            throw new AccountLockedException(
                    "Tài khoản '" + username + "' đã bị khóa. Vui lòng liên hệ Admin.");
        }

        // Bước 5: So khớp mật khẩu
        return user.password().equals(password);
    }
}
```

---

## IV. JUnit 5 Test Class (Java)

```java
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;

import static org.junit.jupiter.api.Assertions.*;

/**
 * Unit Test cho class UserAuthentication.
 * Bao phủ 5 kịch bản kiểm thử: Happy Path, Wrong Password, Locked Account,
 * SQL Injection và Empty Password.
 */
class UserAuthenticationTest {

    private UserAuthentication authService;

    @BeforeEach
    void setUp() {
        // Khởi tạo service mới trước mỗi test để đảm bảo tính độc lập
        authService = new UserAuthentication();
    }

    @Test
    @DisplayName("Kịch bản 1 - Happy Path: Đăng nhập thành công với thông tin hợp lệ")
    void testLoginSuccess_ValidCredentials() {
        // Arrange: Sử dụng user "nguyen.vana" với password đúng
        String username = "nguyen.vana";
        String password = "SecurePass@123";

        // Act
        boolean result = authService.login(username, password);

        // Assert: Đăng nhập phải thành công
        assertTrue(result,
                "Đăng nhập phải thành công khi username và password đều hợp lệ");
    }

    @Test
    @DisplayName("Kịch bản 2 - Wrong Password: Đăng nhập thất bại khi sai mật khẩu")
    void testLoginFailed_WrongPassword() {
        // Arrange: Username đúng nhưng password sai
        String username = "tran.vanb";
        String password = "WrongPassword!456";

        // Act
        boolean result = authService.login(username, password);

        // Assert: Đăng nhập phải thất bại
        assertFalse(result,
                "Đăng nhập phải thất bại khi mật khẩu không khớp");
    }

    @Test
    @DisplayName("Kịch bản 3 - Locked Account: Ném AccountLockedException khi tài khoản bị khóa")
    void testLoginFailed_AccountLocked() {
        // Arrange: Tài khoản "le.vanc" bị khóa (isLocked = true)
        String username = "le.vanc";
        String password = "CorrectPass@789";

        // Act & Assert: Phải ném AccountLockedException
        UserAuthentication.AccountLockedException exception =
                assertThrows(UserAuthentication.AccountLockedException.class,
                        () -> authService.login(username, password),
                        "Phải ném AccountLockedException khi tài khoản bị khóa");

        // Kiểm tra message chứa tên user bị khóa
        assertTrue(exception.getMessage().contains("le.vanc"),
                "Thông báo lỗi phải chứa tên tài khoản bị khóa");
    }

    @Test
    @DisplayName("Kịch bản 4 - SQL Injection: Ném IllegalArgumentException khi username chứa mã độc")
    void testLoginFailed_SqlInjectionDetected() {
        // Arrange: Username chứa chuỗi SQL Injection
        String username = "' OR '1'='1";
        String password = "anything";

        // Act & Assert: Phải ném IllegalArgumentException
        IllegalArgumentException exception =
                assertThrows(IllegalArgumentException.class,
                        () -> authService.login(username, password),
                        "Phải ném IllegalArgumentException khi phát hiện SQL Injection");

        // Kiểm tra message thông báo về ký tự không hợp lệ
        assertTrue(exception.getMessage().toLowerCase().contains("không hợp lệ")
                        || exception.getMessage().toLowerCase().contains("sql injection"),
                "Thông báo lỗi phải đề cập đến ký tự không hợp lệ hoặc SQL Injection");
    }

    @Test
    @DisplayName("Kịch bản 5 - Empty Password: Ném IllegalArgumentException khi mật khẩu rỗng")
    void testLoginFailed_EmptyPassword() {
        // Arrange: Password là chuỗi rỗng
        String username = "pham.vand";
        String password = "";

        // Act & Assert: Phải ném IllegalArgumentException
        IllegalArgumentException exception =
                assertThrows(IllegalArgumentException.class,
                        () -> authService.login(username, password),
                        "Phải ném IllegalArgumentException khi password rỗng");

        // Kiểm tra message thông báo về password trống
        assertTrue(exception.getMessage().toLowerCase().contains("password")
                        || exception.getMessage().toLowerCase().contains("mật khẩu"),
                "Thông báo lỗi phải đề cập đến password không được để trống");
    }
}
```

---

## V. Tổng Kết Bảng Kịch Bản Test

| # | Kịch bản         | Username          | Password            | Expected Result          | Assertion             |
|---|-------------------|-------------------|----------------------|--------------------------|-----------------------|
| 1 | Happy Path        | `nguyen.vana`     | `SecurePass@123`     | `true` (đăng nhập thành công) | `assertTrue`     |
| 2 | Wrong Password    | `tran.vanb`       | `WrongPassword!456`  | `false` (đăng nhập thất bại)  | `assertFalse`    |
| 3 | Locked Account    | `le.vanc`         | `CorrectPass@789`    | `AccountLockedException`      | `assertThrows`   |
| 4 | SQL Injection     | `' OR '1'='1`     | `anything`           | `IllegalArgumentException`    | `assertThrows`   |
| 5 | Empty Password    | `pham.vand`       | `""`                 | `IllegalArgumentException`    | `assertThrows`   |
