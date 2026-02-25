# Leave-Management-Validation
import java.time.LocalDate;
import java.util.*;

// ================= ENUMS =================

enum LeaveType {
    CASUAL, SICK, EARNED
}

enum LeaveStatus {
    PENDING, APPROVED, REJECTED
}

// ================= EXCEPTIONS =================

class InvalidLeaveRequestException extends RuntimeException {
    public InvalidLeaveRequestException(String msg) { super(msg); }
}

class InsufficientLeaveBalanceException extends RuntimeException {
    public InsufficientLeaveBalanceException(String msg) { super(msg); }
}

class EmployeeNotFoundException extends RuntimeException {
    public EmployeeNotFoundException(String msg) { super(msg); }
}

// ================= EMPLOYEE HIERARCHY =================

abstract class Employee {
    private String employeeId;
    private String name;
    protected int leaveBalance;
    protected int totalLeavesTaken;

    public Employee(String employeeId, String name, int leaveBalance) {
        this.employeeId = employeeId;
        this.name = name;
        this.leaveBalance = leaveBalance;
        this.totalLeavesTaken = 0;
    }

    public String getEmployeeId() { return employeeId; }
    public int getLeaveBalance() { return leaveBalance; }
    public int getTotalLeavesTaken() { return totalLeavesTaken; }

    public void deductLeave(int days) {
        leaveBalance -= days;
        totalLeavesTaken += days;
    }

    public abstract int calculateMaxAllowedLeave();
}

class PermanentEmployee extends Employee {
    public PermanentEmployee(String id, String name, int balance) {
        super(id, name, balance);
    }

    public int calculateMaxAllowedLeave() { return 20; }
}

class ContractEmployee extends Employee {
    public ContractEmployee(String id, String name, int balance) {
        super(id, name, balance);
    }

    public int calculateMaxAllowedLeave() { return 10; }
}

// ================= LEAVE REQUEST =================

class LeaveRequest {
    private String employeeId;
    private LeaveType leaveType;
    private int numberOfDays;
    private String reason;
    private LocalDate requestDate;
    private LeaveStatus status;

    public LeaveRequest(String employeeId, LeaveType leaveType,
                        int numberOfDays, String reason) {

        this.employeeId = employeeId;
        this.leaveType = leaveType;
        this.numberOfDays = numberOfDays;
        this.reason = reason;
        this.requestDate = LocalDate.now();
        this.status = LeaveStatus.PENDING;
    }

    public String getEmployeeId() { return employeeId; }
    public LeaveType getLeaveType() { return leaveType; }
    public int getNumberOfDays() { return numberOfDays; }
    public String getReason() { return reason; }
    public LeaveStatus getStatus() { return status; }

    public void setStatus(LeaveStatus status) {
        this.status = status;
    }
}

// ================= POLICY INTERFACE =================

interface LeavePolicy {
    void validateLeave(Employee employee, LeaveRequest request);
}

class CasualLeavePolicy implements LeavePolicy {
    public void validateLeave(Employee emp, LeaveRequest req) {
        if (req.getNumberOfDays() > 5)
            throw new InvalidLeaveRequestException("Casual leave max 5 days.");
    }
}

class SickLeavePolicy implements LeavePolicy {
    public void validateLeave(Employee emp, LeaveRequest req) {
        if (req.getNumberOfDays() > 7)
            throw new InvalidLeaveRequestException("Sick leave max 7 days.");
    }
}

class EarnedLeavePolicy implements LeavePolicy {
    public void validateLeave(Employee emp, LeaveRequest req) {
        // No special rule
    }
}

// ================= REPOSITORIES =================

class EmployeeRepository {
    private Map<String, Employee> employeeMap = new HashMap<>();

    public void addEmployee(Employee e) {
        employeeMap.put(e.getEmployeeId(), e);
    }

    public Employee getEmployeeById(String id) {
        return employeeMap.get(id);
    }
}

class LeaveRepository {
    private List<LeaveRequest> leaveList = new ArrayList<>();

    public void saveLeave(LeaveRequest request) {
        leaveList.add(request);
    }
}

// ================= SERVICE =================

class LeaveService {

    private EmployeeRepository employeeRepo;
    private LeaveRepository leaveRepo;

    public LeaveService(EmployeeRepository eRepo, LeaveRepository lRepo) {
        this.employeeRepo = eRepo;
        this.leaveRepo = lRepo;
    }

    public void applyLeave(LeaveRequest request) {

        Employee employee =
                employeeRepo.getEmployeeById(request.getEmployeeId());

        if (employee == null)
            throw new EmployeeNotFoundException("Employee not found.");

        LeavePolicy policy = getPolicy(request.getLeaveType());

        // 1. Policy validation
        policy.validateLeave(employee, request);

        // 2. Annual max validation
        if (employee.getTotalLeavesTaken() + request.getNumberOfDays()
                > employee.calculateMaxAllowedLeave())
            throw new InvalidLeaveRequestException("Annual limit exceeded.");

        // 3. Balance validation
        if (request.getNumberOfDays() > employee.getLeaveBalance())
            throw new InsufficientLeaveBalanceException("Insufficient balance.");

        // 4. Deduct + approve
        employee.deductLeave(request.getNumberOfDays());
        request.setStatus(LeaveStatus.APPROVED);

        leaveRepo.saveLeave(request);
    }

    private LeavePolicy getPolicy(LeaveType type) {
        switch (type) {
            case CASUAL: return new CasualLeavePolicy();
            case SICK: return new SickLeavePolicy();
            case EARNED: return new EarnedLeavePolicy();
            default: throw new InvalidLeaveRequestException("Invalid type");
        }
    }
}

// ================= MAIN =================

public class Main {

    public static void main(String[] args) {

        EmployeeRepository empRepo = new EmployeeRepository();
        LeaveRepository leaveRepo = new LeaveRepository();
        LeaveService service = new LeaveService(empRepo, leaveRepo);

        empRepo.addEmployee(new PermanentEmployee("E1", "Alice", 15));
        empRepo.addEmployee(new ContractEmployee("E2", "Bob", 8));

        try {
            LeaveRequest request =
                new LeaveRequest("E1", LeaveType.CASUAL, 3, "Vacation");

            service.applyLeave(request);

            System.out.println("Leave Status: " + request.getStatus());

        } catch (Exception e) {
            System.out.println("Error: " + e.getMessage());
        }
    }
}
