
StatusNotificationListener - a Service Listener who got notified when a status change occur
has notify method which called by DAO
and boolean shouldEmail uses predicate to note if lisner needs to fire an email
has member field EmailService
has member field EmailProperties

EmailService - a Service responsible for emailing using a thymeleaf template. Before sending email
convert the template to a valid html using templateService
has sendEmail(EmailRequest request)
has member field TemplateService

TemplateService - a Service responsible for converting a map to an HTML String
has renderTemplate(String templateName, Map<String, Object> model)
has member field TemplateEngine of Thymeleaf injected by Spring

EmailRequest Pojo has
emailTo List<String>
emailCc List<String>
emailFrom String
emailSubject String
emailTemplate String
model Map<String, Object>

I have these Notification Rules

| Role             | Status                     | What field / how to get recipients      |
|------------------|----------------------------|-----------------------------------------|
| Job Manager      | Awaiting Approval          | JobManager field in Entity              |
| Job Manager      | Awaiting Regional Approval | JobManager field in Entity              |
| Regional Manager | Awaiting Regional Approval | Regional Manager field in Entity        |
| Planning Group   | Awaiting Endorsement       | Distribution group from config          |
| Requestor        | Awaiting Revision          | Owner field in Entity                   |
| Requestor        | Withdrawn                  | Owner field in Entity                   |
| Requestor        | Completed                  | Owner field in Entity                   |
| Service Provider | Completed                  | Distribution group from config          |
| Service Provider | Withdrawn                  | Distribution group from config          |


Status transitions
Draft > Submit > Awaiting Endorsement > Awaiting Approval > Complete
Draft > Submit > Awaiting Endorsement > Awaiting Approval > Awaiting Regional Approval > Complete
Draft > Submit > Awaiting Approval > Complete
Draft > Submit > Awaiting Endorsement > Awaiting Revision >  Withdrawn > Draft

Email Config EmailProperties
region:
  north: north-notify@domain.com
  south: south-notify@domain.com
planning:
  north: north-planning-notify@domain.com
  south: south-planning-notify@domain.com
service-provider:
  north: north-svc-notify@domain.com
  south: south-svc-notify@domain.com  


1. How can i create a NotificationListeners and add the Notification Rules that can be extensible and tested in isolation.
2. Is it possible to have a generic StatusUpdateLister then a custom for each Role e.g. JobManagerNotificationListener  

A. Define the Notification Strategy Interface (The "Rule")

The best way to handle your notification rules is to use the Strategy Pattern or similar techniques like a Chain of Responsibility or a Rule Engine, but a centralized, rule-based Strategy Pattern is usually the most straightforward for this kind of lookup table.

A. Define the Notification Strategy Interface (The "Rule")
Create an interface that represents a single notification rule.

public interface NotificationRule {
    // A predicate to determine if this specific rule should execute
    // based on the context (the entity and the new status).
    boolean appliesTo(Entity entity, Status newStatus);

    // Determines the recipients (To, CC) for this rule.
    // It is responsible for fetching users from the entity/context or config.
    EmailRecipients getRecipients(Entity entity, EmailProperties properties);

    // Defines the email template to be used for this rule.
    String getTemplateName();

    // Defines the subject for this rule.
    String getSubject();
}

// Helper class to hold To and CC lists
public class EmailRecipients {
    private List<String> to;
    private List<String> cc;
    // + constructor, getters/setters
}


B. Implement Concrete Rule Classes
Each row in your rule table, or a logical grouping of rows, becomes a concrete implementation of NotificationRule.

@Component
public class JobManagerAwaitingApprovalRule implements NotificationRule {

    @Override
    public boolean appliesTo(Entity entity, Status newStatus) {
        // Matches the "Job Manager | Awaiting Approval" rule
        return newStatus == Status.AWAITING_APPROVAL;
    }

    @Override
    public EmailRecipients getRecipients(Entity entity, EmailProperties properties) {
        // Assuming entity.getJobManagerEmail() returns the email from the entity field
        return new EmailRecipients(
            Collections.singletonList(entity.getJobManagerEmail()),
            Collections.emptyList()
        );
    }

    @Override
    public String getTemplateName() {
        return "job-manager-awaiting-approval.html";
    }

    @Override
    public String getSubject() {
        return "Action Required: Job is Awaiting Your Approval";
    }
}

@Component
public class ServiceProviderRule implements NotificationRule {

    @Override
    public boolean appliesTo(Entity entity, Status newStatus) {
        // Matches the Service Provider for Completed or Withdrawn
        return newStatus == Status.COMPLETED || newStatus == Status.WITHDRAWN;
    }

    @Override
    public EmailRecipients getRecipients(Entity entity, EmailProperties properties) {
        // Fetches distribution group from config based on a region field in the entity
        String regionKey = entity.getRegion().toLowerCase(); // e.g., "north"
        String email = properties.getServiceProvider().get(regionKey);

        return new EmailRecipients(
            Collections.singletonList(email),
            Collections.emptyList()
        );
    }
    // ... other methods
}

The Centralized NotificationManager
Instead of a single StatusNotificationListener, introduce a NotificationRuleManager service. This manager collects all NotificationRule implementations and processes them.

@Service
public class NotificationRuleManager {

    private final List<NotificationRule> rules;
    private final EmailService emailService;
    private final EmailProperties emailProperties;

    // Spring will automatically inject ALL beans that implement NotificationRule
    public NotificationRuleManager(
        List<NotificationRule> rules,
        EmailService emailService,
        EmailProperties emailProperties
    ) {
        this.rules = rules;
        this.emailService = emailService;
        this.emailProperties = emailProperties;
    }

    public void processStatusUpdate(Entity entity, Status newStatus) {
        // For each rule, check if it applies.
        rules.stream()
            .filter(rule -> rule.appliesTo(entity, newStatus))
            .forEach(rule -> {
                // If the rule applies, execute the email logic
                EmailRecipients recipients = rule.getRecipients(entity, emailProperties);

                EmailRequest request = new EmailRequest();
                request.setEmailTo(recipients.getTo());
                request.setEmailCc(recipients.getCc());
                // Set other common fields (From, Model data, Subject, Template Name)
                request.setEmailSubject(rule.getSubject());
                request.setEmailTemplate(rule.getTemplateName());
                request.setModel(createModel(entity)); // Utility method to build the template model

                emailService.sendEmail(request);
            });
    }

    // Helper method to build the Map<String, Object> for the Thymeleaf template
    private Map<String, Object> createModel(Entity entity) {
        // ... put entity data into a map for the template
        return Map.of("jobName", entity.getName(), "entity", entity);
    }
}

@Service
public class StatusNotificationListener {

    private final NotificationRuleManager ruleManager;

    public StatusNotificationListener(NotificationRuleManager ruleManager) {
        this.ruleManager = ruleManager;
    }

    // This is the method called by your DAO/Service
    public void notify(Entity entity, Status oldStatus, Status newStatus) {
        // Optional: Implement the predicate logic here if it's generic
        // or move it into the rule manager.
        // boolean shouldEmail = ... (e.g., checks if status changed)

        ruleManager.processStatusUpdate(entity, newStatus);
    }
}


--- Testing

import org.junit.jupiter.params.provider.Arguments;
import java.util.stream.Stream;

// Define a placeholder Entity class for testing
class TestEntity {
    private Status status;
    private String jobManagerEmail;
    private String region;
    // + constructor, getters for all fields used by rules
}

public class JobManagerAwaitingApprovalRuleTest {

    // 1. The Method Provider
    private static Stream<Arguments> ruleTestCases() {
        // Mock Entity setup
        TestEntity entity = new TestEntity(
            Status.AWAITING_APPROVAL, 
            "job.manager@company.com", 
            "north"
        );

        // Define expected recipients for this entity
        EmailRecipients expectedRecipients = new EmailRecipients(
            List.of("job.manager@company.com"), 
            Collections.emptyList()
        );

        // Define EmailProperties (needed for rules that fetch from config)
        EmailProperties properties = new EmailProperties(); 
        // Populate properties with test data if necessary, though not strictly needed 
        // for this specific rule as it uses the entity field.

        return Stream.of(
            Arguments.of(
                entity, 
                Status.AWAITING_APPROVAL, // newStatus
                properties,
                true, // Expected result for appliesTo(entity, newStatus)
                expectedRecipients, // Expected recipients
                "job-manager-awaiting-approval.html", // Expected template name
                "Action Required: Job is Awaiting Your Approval" // Expected subject
            ),
            // Add a negative case to ensure it doesn't apply when it shouldn't
            Arguments.of(
                entity, 
                Status.COMPLETED, // newStatus
                properties,
                false, // Expected result for appliesTo
                null, // Recipients not relevant for false case
                null, // Template not relevant
                null // Subject not relevant
            )
        );
    }
    
    // ... rest of the test class below
}

import org.junit.jupiter.api.Assertions;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.MethodSource;

public class JobManagerAwaitingApprovalRuleTest {
    
    // (A. The ruleTestCases method is here)

    @ParameterizedTest(name = "[{index}] Status:{1}, Applies:{3}")
    @MethodSource("ruleTestCases")
    void testRule_whenGivenStatusAndEntity_returnsExpectedValues(
        TestEntity entity,
        Status newStatus,
        EmailProperties properties,
        boolean expectedAppliesTo,
        EmailRecipients expectedRecipients,
        String expectedTemplateName,
        String expectedSubject
    ) {
        // GIVEN
        JobManagerAwaitingApprovalRule rule = new JobManagerAwaitingApprovalRule();

        // 1. Test appliesTo()
        Assertions.assertEquals(expectedAppliesTo, rule.appliesTo(entity, newStatus), 
            "The appliesTo() predicate failed.");

        if (expectedAppliesTo) {
            // 2. Test getRecipients()
            EmailRecipients actualRecipients = rule.getRecipients(entity, properties);
            Assertions.assertEquals(expectedRecipients.getTo(), actualRecipients.getTo(), 
                "The 'To' recipients list failed.");
            Assertions.assertEquals(expectedRecipients.getCc(), actualRecipients.getCc(), 
                "The 'CC' recipients list failed.");
            
            // 3. Test getTemplateName()
            Assertions.assertEquals(expectedTemplateName, rule.getTemplateName(), 
                "The template name failed.");
                
            // 4. Test getSubject()
            Assertions.assertEquals(expectedSubject, rule.getSubject(), 
                "The subject failed.");
        }
    }
}

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mockito.ArgumentCaptor;
import org.mockito.Mockito;
import static org.mockito.Mockito.*;

public class NotificationRuleManagerTest {

    private NotificationRuleManager ruleManager;
    private EmailService emailService;
    private EmailProperties emailProperties;
    private List<NotificationRule> rules;

    // Mock rule implementations for testing
    private NotificationRule mockRuleA;
    private NotificationRule mockRuleB;

    @BeforeEach
    void setUp() {
        emailService = mock(EmailService.class);
        emailProperties = new EmailProperties(); // Use a real or simple mock
        
        mockRuleA = mock(NotificationRule.class);
        mockRuleB = mock(NotificationRule.class);
        rules = List.of(mockRuleA, mockRuleB);
        
        // Initialize the manager with the mock dependencies
        ruleManager = new NotificationRuleManager(rules, emailService, emailProperties);
    }

    @Test
    void processStatusUpdate_callsEmailServiceOnlyForApplicableRules() {
        // GIVEN
        TestEntity entity = new TestEntity(Status.AWAITING_APPROVAL, "owner@company.com", "north");
        Status newStatus = Status.AWAITING_APPROVAL;
        
        // Setup Mocks:
        // Rule A: APPLIES
        when(mockRuleA.appliesTo(entity, newStatus)).thenReturn(true);
        when(mockRuleA.getRecipients(entity, emailProperties))
            .thenReturn(new EmailRecipients(List.of("ruleA-to@company.com"), List.of("ruleA-cc@company.com")));
        when(mockRuleA.getTemplateName()).thenReturn("template-A");
        when(mockRuleA.getSubject()).thenReturn("Subject A");

        // Rule B: DOES NOT APPLY
        when(mockRuleB.appliesTo(entity, newStatus)).thenReturn(false);

        // WHEN
        ruleManager.processStatusUpdate(entity, newStatus);

        // THEN
        // 1. Verify sendEmail was called exactly once (only for Rule A)
        verify(emailService, times(1)).sendEmail(any(EmailRequest.class));
        
        // 2. Capture and verify the content of the EmailRequest sent
        ArgumentCaptor<EmailRequest> emailRequestCaptor = ArgumentCaptor.forClass(EmailRequest.class);
        verify(emailService).sendEmail(emailRequestCaptor.capture());

        EmailRequest sentRequest = emailRequestCaptor.getValue();
        
        Assertions.assertEquals(List.of("ruleA-to@company.com"), sentRequest.getEmailTo(), "Recipients must match Rule A.");
        Assertions.assertEquals("template-A", sentRequest.getEmailTemplate(), "Template must match Rule A.");
        Assertions.assertEquals("Subject A", sentRequest.getEmailSubject(), "Subject must match Rule A.");

        // 3. Verify Rule B's methods were never called after appliesTo
        verify(mockRuleB, never()).getRecipients(any(), any());
    }
}


----- plant UML

@startuml
title Status Notification System Flow

actor "Request Initiator (User/System)" as User
participant "DAO/Repository" as DAO
participant "EntityService" as Service #lightblue
participant "StatusUpdateEvent" as Event #pink
participant "StatusNotificationListener" as Listener #green
participant "NotificationRuleManager" as Manager #green
box "Notification Rules (Strategy Pattern)" #LightYellow
    participant "RuleA: JobManagerApprovalRule" as RuleA
    participant "RuleB: RequestorRevisionRule" as RuleB
end box
participant "EmailService" as MailSvc #orange
participant "TemplateService" as TplSvc #darkgrey
database "EmailProperties (Config)" as Config

== 1. Entity Update and Event Publishing ==

User -> Service: updateStatus(entityId, newStatus)
Service -> DAO: find/load(entityId)
DAO --> Service: Entity
Service -> DAO: save(Entity)
DAO --> Service: Updated Entity
Service -> Event: publish(StatusUpdateEvent)

== 2. Event Handling and Rule Processing ==

Event -> Listener: handleStatusUpdate(event)
Listener -> Manager: processStatusUpdate(entity, newStatus)

Manager -> Manager: createModel(entity)

loop For each NotificationRule in list
    Manager -> RuleA: appliesTo(entity, newStatus)
    alt RuleA Applies (e.g., Awaiting Approval)
        RuleA --> Manager: true
        Manager -> RuleA: getRecipients(entity, Config)
        RuleA -> Config: lookup distribution list (if needed)
        Config --> RuleA: distributionEmail
        RuleA --> Manager: EmailRecipients(To, CC)

        Manager -> RuleA: getTemplateName()
        RuleA --> Manager: templateName

        Manager -> MailSvc: sendEmail(EmailRequest)

        MailSvc -> TplSvc: renderTemplate(templateName, model)
        TplSvc -> TplSvc: load template from file/cache
        TplSvc --> MailSvc: htmlContent
        MailSvc -> MailSvc: assemble email (subject, body)
        MailSvc -> "External SMTP" : transmit email

    else RuleA Does Not Apply
        RuleA --> Manager: false
    end

    Manager -> RuleB: appliesTo(entity, newStatus)
    alt RuleB Applies (e.g., Awaiting Revision)
        RuleB --> Manager: true
        Manager -> RuleB: getRecipients(entity, Config)
        RuleB --> Manager: EmailRecipients(To, CC)
        
        Manager -> RuleB: getTemplateName()
        RuleB --> Manager: templateName
        
        Manager -> MailSvc: sendEmail(EmailRequest)
        ' Subsequent steps handled by MailSvc as above
    end
end
Manager --> Listener: complete
Listener --> Service: complete
Service --> User: Entity

@enduml

Key Interactions Explained:
Decoupling with Events: The EntityService is decoupled from the notification logic using the StatusUpdateEvent. The Service simply updates the entity and publishes an event; it doesn't know who is listening or what happens next.

Listener as Gateway: The StatusNotificationListener acts as the subscriber, receiving the event and delegating the core logic to the NotificationRuleManager.

Strategy Pattern: The NotificationRuleManager is responsible for iterating over all available strategies (RuleA, RuleB, etc.) and checking which one is applicable using the appliesTo() method.

Rule Execution: If a rule applies, it executes its logic (getRecipients, getTemplateName) to build the necessary data for the email.

Email Generation: The EmailService receives a fully formed EmailRequest and relies on the TemplateService to convert the template name and model data into the final HTML body before sending the message.