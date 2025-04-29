# Report

## 1. Пример кода для запроса данных из бд, формирования отчета и отправки по почте

```java

public class SalaryHtmlReportNotice {

    private static final Logger logger = LoggerFactory.getLogger(???);
    private ....

    public void generateAndSendReport() {
        Connection connection = null;

        try {
            connection = DriverManager.getConnection("jdbc:..., "user", "password");

            // Условно псевдозапрос по памяти
            String sql = "SELECT e.id AS userID, SUM(d.salary) AS salary_sum " +
                    "FROM employee e " +
                    "JOIN department d ON e.department_id = d.id " +
                    "WHERE d.date >= ? AND d.name = ? " +
                    "GROUP BY e.id";

            PreparedStatement ps = connection.prepareStatement(sql);
            ps.setDate(...);
            ps.setString(....);
            ...

            ResultSet result = ps.executeQuery();

            StringBuilder html = new StringBuilder();
            html.append("<html><body><table ='...'>");
            html.append("<tr><th>Имя</th><th>зарплата</th></tr>");
            while (result.next()) {
                String userID = result.getString("userID");
                double salarySum = result.getDouble("salary_sum");

                html.append("<tr>")
                        .append("<td>").append(userID).append("</td>")
                        .append("<td>").append(salarySum).append("</td>")
                        .append("</tr>");
            }
            html.append("</table></body></html>");

            Properties props = new Properties();
            props.put("mail.smtp.host", "...");
            props.put("mail.smtp.port", "...");

            Session session = Session.getInstance(props);
            Message message = new MimeMessage(session);
            message.setFrom(("...@...ru"));
            message.setRecipients(Message...);
            message.setSubject("Отчет...");
            message.setContent(html.., "text/html");
            Transport.send(message);

        } catch (SQLException e) {
            logger.error("Ошибка с БД...: " + e.getMessage(), e);
        } catch (MessagingException e) {
            logger.error("Ошибка при отправке: " + e.getMessage(), e);
        } finally {
            if (connection != null) {
                try {
                    connection.close();
                } catch ...
            }
        }
    }
}
```

## 2. Идеи рефакторинга кода
### 1) SalaryHtmlReportNotice => SalaryReportNotice
Нужен отдельный сервис, в котором можно генерировать отчет в нужный формат и отправлять уведомление на почту в виде html или в другом формате.

```java
@Service
public class SalaryReportNoticeServiceImpl implements SalaryReportNoticeService {

    private static final Logger logger = LoggerFactory.getLogger(SalaryReportNotice.class);

    private final EmployeeRepository employeeRepository; // отдельный сервис для запросов к бд
    private final ReportFormatService reportFormatService; // отдельный сервис для аггрегации данных и преобразования в нужный формат
    private final MailService mailService; // отдельный почтовый сервис

    @Autowired
    public SalaryReportNoticeServiceImpl(
            EmployeeRepository employeeRepository,
            ReportFormatService reportFormatService,
            MailService mailService
    ) {
        this.employeeRepository = employeeRepository;
        this.reportFormatService = reportFormatService;
        this.mailService = mailService;
    }

    /**
     * Генерирует отчет по зарплатам сотрудников выбранной орагнизации за заданный период и отправляет его по электронной почте.
     *
     * @param departmentName название организации, по которому нужно сформировать отчет
     * @param date дата для периода
     * @param reportFormat формат отчета:  "html", "json", "xml"
     */
    public void generateAndSendReport(String departmentName, Date date, String reportFormat) {
        try {
            logger.info("Начинаем генерацию отчета для : {} и даты: {}", departmentName, date);

            // Получаем данные из запроса
            List<EmployeeSalaryDTO> results = employeeRepository.findEmployeeSalarySumByDepartmentAndDate(departmentName, date);
            logger.info("Получено {} записей по запросу для департамента: {}", results.size(), departmentName);

            if (results.isEmpty()) {
                logger.error("Нет данных для отчета. Отчет не будет отправлен.");
                return;
            }

            // Формируем отчет в нужном формате
            String report = formatReport(results, reportFormat);
            logger.info("Отчет успешно сформирован.");

            // Отправляем отчет по email
            mailService.sendEmail("test@test.ru", "Отчет...", report);
            logger.info("Отчет отправлен на email test@test.ru.");

        } catch (Exception e) {
            logger.error("Ошибка при формировании отчета: {}", e.getMessage(), e);
        }
    }
    // Формируем отчет в нужный формат. Подумал в сторону абстракции передавая например в аргумент тип желаемоего формата и на его основании использовать паттерн-матчинг и отправлять в нужном формате 
    private String formatReport(List<EmployeeSalaryDTO> results, String reportFormat) {
        switch (reportFormat.toLowerCase()) {
            case "html":
                return reportFormatService.formatToHtml(results);
            case "json":
                return reportFormatService.formatToJson(results);
            case "xml":
                return reportFormatService.formatToXml(results);
            default:
                throw new IllegalArgumentException("Неизвестный формат отчета: " + reportFormat);
        }
    }

}
```


### 2) ДТО и Модели
TODO уточнение полей и целой структуры
```java
@Entity
@Data
public class Employee {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne
    @JoinColumn(name = "department_id")
    private Department department;

    private Double salary;
}
```

```java
@Entity
@Data
public class Department {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    @Temporal(TemporalType.DATE)
    private Date date;
}
```

```java
@Data
@NoArgsConstructor
@AllArgsConstructor
public class EmployeeSalaryDTO {
    private Long employeeId;
    private Double totalSalary;
}
```

### 3) Универсальный парсер с масштабированием
TODO 
```java

public interface ReportFormatterUtils {

    String formatToHtml(List<EmployeeSalaryDTO> results);

    String formatToJson(List<EmployeeSalaryDTO> results);

    String formatToXml(List<EmployeeSalaryDTO> results);
}

@Service
public class ReportFormatServiceImpl implements ReportFormatterUtils {

    @Override
    public String formatToHtml(List<EmployeeSalaryDTO> results) {
        StringBuilder html = new StringBuilder();
        html.append("<html><body><table>");
        html.append("<tr><th>Имя</th><th>Зарплата</th></tr>");
        for (EmployeeSalaryDTO result : results) {
            // TODO доп бизнес-логика с вычислениями
            html.append("<tr>")
                    .append("<td>").append(result.getUserId()).append("</td>")
                    .append("<td>").append(result.getSalarySum()).append("</td>")
                    .append("</tr>");
        }
        html.append("</table></body></html>");
        return html.toString();
    }

    @Override
    public String formatToJson(List<EmployeeSalaryDTO> results) {
        StringBuilder json = new StringBuilder();
        json.append("[");
        for (int i = 0; i < results.size(); i++) {
            // TODO доп бизнес-логика с вычислениями
            EmployeeSalaryDTO result = results.get(i);
            json.append("{")
                    .append("\"userId\":").append(result.getUserId()).append(", ")
                    .append("\"salarySum\":").append(result.getSalarySum())
                    .append("}");
            if....
        }
        json.append("]");
        return json.toString();
    }

    @Override
    public String formatToXml(List<EmployeeSalaryDTO> results) {
        StringBuilder xml = new StringBuilder();
        xml.append("<report>");
        for (EmployeeSalaryDTO result : results) {
            // TODO доп бизнес-логика с вычислениями
            xml.append("<entry>")
                    .append("<userId>").append(result.getUserId()).append("</userId>")
                    .append("<salarySum>").append(result.getSalarySum()).append("</salarySum>")
                    .append("</entry>");
        }
        xml.append("</report>");
        return xml.toString();
    }
}
```

### 4) Репо
TODO 
```java
@Repository
public interface EmployeeRepository extends JpaRepository<Employee, Long> {
    
    // TODO вопрос каким вариантом писать запрос
    @Query("SELECT (e.id, SUM(e.salary)) " +
           "FROM Employee e " +
           "JOIN e.department d " +
           "WHERE d.name = :departmentName AND d.date >= :date " +
           "GROUP BY e.id")
    List<EmployeeSalaryDTO> findEmployeeSalarySumByDepartmentAndDate(@Param("departmentName") String departmentName, @Param("date") Date date);
}

```

### 5) Почтовый сервис
TODO 
```java
@Service
public class MailService {

    private static final Logger logger = LoggerFactory.getLogger(SalaryReportNotice.class);

    public void sendEmail(String to, String subject, String body, String path) {

        Properties props = new Properties();

        props.put("mail.smtp.host", "...");       
        props.put("mail.smtp.port", "...");       
        props.put("mail.smtp.auth", "true");     
        props.put("mail.smtp.starttls.enable", "true");

        Session session = Session....});

        try {
            Message msg = new MimeMessage(session);
            msg.setFrom(new InternetAddress("..."));
            msg.setRecipients(...);
            msg.setSubject(subject);

            MimeBodyPart htmlPart = new MimeBodyPart();
            htmlPart.setContent(body, "text/html");

            Multipart multipart = new MimeMultipart();
            multipart.addBodyPart(htmlPart);

            if (path != null && !path.isEmpty()) {
                MimeBodyPart filePart = new MimeBodyPart();
                File file = new File(attachmentPath);
                if (file.exists()) {
                    DataSource source = new FileDataSource(file);
                    filePart.setDataHandler(new DataHandler(source));
                    filePart.setFileName(file.getName());
                    multipart.addBodyPart(filePart);
                }
            }

            msg.setContent(multipart);
            Transport.send(msg);

        } catch (MessagingException e) {
            logger...
        }
    }
}
```

## 3. Проработка устных вопросов

#### 1) Что такое "Транзакционность"?
   вначале подумал про связанно понятие ACID (атомарность, консистентность, изоляция, надежность)
   
   Позже на интервью я вспомнил и прокомментировал, что в спринге есть аннотация @Transactional, которая используется для автоматического управления транзакциями.
   
   Дома уже я разобрал основной прицнип. Помечая метод такой аннотацией, мы гарантируем, что метод либо выполнится полностью, либо откатит все изменения, если будет ошибка в операции. 
   
   Все изменения, сделанные в базе в рамках этой транзакции, будут уже отменены. На последней работе я работаю с паттерном-Saga, библиотека для транзакций.
#### 2) Назвать какие джоины я знаю. 
Я назвал 3, но забыл про четвертный, это FULL JOIN. Возможность получить все строки из таблиц, если не совпадают по полю или условию
![image](https://github.com/user-attachments/assets/b7358080-629b-4475-9755-62c74762ca66)

#### 3) Отличие JPA и Hibernate
JPA является стандартом для работы с базой данных, описывающим API для сохранения данных, а Hibernate — это реализация этого стандарта с дополнительными возможностями для работы с ORM

#### 4) Про интерфейсы коллекции, которые знаю в джава 
Вчера и сегодня повторил базовые типы и коллекции в джава. 
В Java есть коллекция (Collection) как базовый интерфейс и коллекции интерфейсов List, Set, Queue и Map

Интерфейс -> Реализация:

List -> ArrayList, LinkedList, Vector, Stack

Set -> HashSet, LinkedHashSet, TreeSet

Map -> HashMap, LinkedHashMap, TreeMap, Hashtable, ConcurrentHashMap

Queue -> LinkedList, PriorityQueue, ArrayDeque, Deque

#### 5) Как устроен HashMap и что будет если хеши совпадают

HashMap использует массив для хранения данных. Данные в массиве записаны как ключ- значение. Также если правильно помню, вчера говорил, что доступ к мапе достаточно быстрый за счет хеша.

Из особенностей, что ключи должны быть уникальными, а значения могут дублироваться. 
Если происходит совпадение ключей, то происходит коллизия и элементы могут хранятся в виде связанных списков или деревьев

P.S не сразу вспомнил про понятие "экранные функции")). Позже на досуге разобрался и дополнил знания


