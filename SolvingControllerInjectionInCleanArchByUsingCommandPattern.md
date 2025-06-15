# Clean Architecture: Giải quyết Controller Injection dùng Command Pattern

## 🧱 1. **Command Pattern là gì?**

### Định nghĩa:

> Command Pattern là một design pattern thuộc nhóm behavioral, nhằm đóng gói một hành động (yêu cầu) thành một đối tượng riêng biệt.
> 

### Cấu trúc tổng quát:

- `Command`: interface định nghĩa một hành động.
- `ConcreteCommand`: class thực thi hành động.
- `Receiver`: đối tượng thực hiện hành động thực tế.
- `Invoker`: đối tượng gọi `execute()` trên `Command`.
- `Client`: tạo `ConcreteCommand`, gán cho `Invoker`.

### Mục tiêu:

- **Tách biệt** giữa nơi gửi yêu cầu và nơi thực thi yêu cầu.
- Hỗ trợ:
    - Undo/Redo
    - Logging các hành động
    - Queue các command
    - Deferred execution

---

## 🚍 2. **Command Bus là gì?**

### Định nghĩa:

> Command Bus là một cơ chế trung gian để gửi các Command tới CommandHandler tương ứng một cách tự động.
> 

### Mục tiêu:

- Thay vì gọi `commandHandler.handle(command)` trực tiếp, bạn chỉ cần `commandBus.execute(command)`.
- **Command Bus = Dispatcher trung tâm** giúp:
    - Tìm đúng handler
    - Thực thi handler
    - Giao tiếp với DI container (Spring context)

### Các thành phần:

- `Command`: request thay đổi trạng thái.
- `CommandHandler`: xử lý command.
- `CommandBus`: nhận `Command`, tìm `Handler`, gọi `handle`.

## 🔗 3. **Điểm giao giữa Command Pattern và Command Bus**

| Thành phần | Command Pattern | Command Bus |
| --- | --- | --- |
| `Command` interface | ✅ (core của pattern) | ✅ (dùng làm hợp đồng cho dispatching) |
| `CommandHandler` | Thường là `ConcreteCommand` | Là class xử lý business logic tương ứng |
| `Invoker` | Tùy theo cấu trúc cụ thể | ✅ Chính là `CommandBus` |
| `Receiver` | Được handler gọi | ✅ Có thể nằm trong handler |
| Tìm handler tự động | ❌ (thường binding thủ công) | ✅ Có thêm logic như `CommandHandlerResolver` |
| Tích hợp DI (Spring) | ❌ Không có sẵn | ✅ `CommandBus` dùng Spring context để inject |

✅ Điểm kết hợp là ở:

- `Command` đóng gói hành vi (từ Command Pattern).
- `CommandBus` là "Invoker động" sử dụng dependency injection & reflection để tự động tìm đúng `CommandHandler`.

## 🏗️ 4. Luồng **Compile-time** và **Runtime**

### ✅ Compile-Time:

- Interface `Command<T>` và `CommandHandler<C, R>` được định nghĩa sẵn.
- Annotation `@HandlesCommand` được gắn vào các handler.
- Spring scan và quản lý các bean trong context (nhưng chưa mapping runtime).

### 🔁 Runtime:

Khi gọi `commandBus.execute(command)`:

1. **CommandBus nhận lệnh** → `CreateUserCommand` chẳng hạn.
2. **Dùng `CommandHandlerResolver` (reflection)**:
    - Dò tất cả class có `@HandlesCommand(CreateUserCommand.class)`
    - Trả về class handler tương ứng, ví dụ `CreateUserHandler.class`
3. **Lấy handler từ Spring Context**:
    - `context.getBean(handlerType)`
4. **Gọi `.handle(command)`**
5. **Trả về kết quả**

## 🧪 5. Áp dụng Command Pattern + Command Bus

### ✅ 1. `Command<T>` – Đại diện cho một yêu cầu hành vi

```java
public interface Command<T> {
}
```

Đây là một **marker interface**, đại diện cho các hành động thay đổi trạng thái hệ thống. `T` là kiểu kết quả trả về sau khi command được xử lý.

Ví dụ, ta định nghĩa command tạo người dùng:

```java
public record CreateUserCommand(String username, String password) implements Command<UserDto> {}
```

---

### ✅ 2. `CommandHandler<C, R>` – Bộ xử lý command

```java
public interface CommandHandler<C extends Command<R>, R> {
    R handle(C command);
}
```

Một `CommandHandler` chịu trách nhiệm xử lý **một loại command cụ thể**. 

---

### ✅ 3. `@HandlesCommand` – Kết nối Command và Handler

Đây là một custom annotation giúp ánh xạ một `CommandHandler` với một `Command`:

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface HandlesCommand {
    Class<? extends Command<?>> value();
}
```

Trong ví dụ này`@HandlesCommand` giúp ánh xạ `CreateUserCommandHandler` xử lý `CreateUserCommand` :

```java
@HandlesCommand(CreateUserCommand.class)
public class CreateUserCommandHandler implements CommandHandler<CreateUserCommand, UserDto> {
    ...
}
```

---

### ✅ 4. `CommandHandlerResolver` – Tự động tìm đúng handler

Sử dụng thư viện `Reflections`, ta có thể quét package dự án để tìm đúng `CommandHandler` tương ứng với một `Command`:

```java
@Slf4j
public class CommandHandlerResolver {

    private static final String BASE_PACKAGE = "com.example"; // đổi thành package bạn dùng

    public static Class<? extends CommandHandler<?, ?>> resolve(Class<? extends Command<?>> commandClass) {
        Reflections reflections = new Reflections(BASE_PACKAGE);
        Set<Class<? extends CommandHandler<?, ?>>> handlerClasses =
            reflections.getSubTypesOf(CommandHandler.class);

        for (Class<? extends CommandHandler<?, ?>> handlerClass : handlerClasses) {
            HandlesCommand annotation = handlerClass.getAnnotation(HandlesCommand.class);
            if (annotation != null && annotation.value().equals(commandClass)) {
                return handlerClass;
            }
        }

        throw new RuntimeException("No handler found for command: " + commandClass.getName());
    }
}
```

---

### ✅ 5. `CommandBus` – Trung tâm điều phối command

```java
public interface CommandBus {
    <T> T execute(Command<T> command);
}
```

Triển khai cụ thể bằng Spring:

```java
@Component
public class SpringCommandBus implements CommandBus {

    private final ApplicationContext context;

    public SpringCommandBus(ApplicationContext context) {
        this.context = context;
    }

    @Override
    @SuppressWarnings("unchecked")
    public <T> T execute(Command<T> command) {
        var handlerType = CommandHandlerResolver.resolve(command.getClass());
        var handler = (CommandHandler<Command<T>, T>) context.getBean(handlerType);
        return handler.handle(command);
    }
}
```