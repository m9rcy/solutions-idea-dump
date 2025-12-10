import java.util.function.Function;

/**
 * Represents a single approval step, storing type-safe Method References
 * for each of the three fields we need.
 * T: The Entity type (e.g., Entity).
 * R: The field type (e.g., String).
 */
class FullyGenericApprovalStep<T, R> {
    
    // The getter function for the retained field (used for checking if the step acted)
    private final Function<T, R> retainedFieldGetter; 
    
    // The getter functions for the two other fields
    private final Function<T, R> decidedByGetter;
    private final Function<T, R> decisionGetter;

    /**
     * @param retainedFieldGetter The getter for the persistent field (e.g., comments).
     * @param decidedByGetter The getter for the decidedBy field.
     * @param decisionGetter The getter for the decision field.
     */
    public FullyGenericApprovalStep(Function<T, R> retainedFieldGetter, 
                                     Function<T, R> decidedByGetter, 
                                     Function<T, R> decisionGetter) {
        this.retainedFieldGetter = retainedFieldGetter;
        this.decidedByGetter = decidedByGetter;
        this.decisionGetter = decisionGetter;
    }

    /**
     * Checks if the persistent field (retainedField) is populated on a given entity.
     */
    public boolean hasActed(T entity) {
        R value = retainedFieldGetter.apply(entity);
        // Assuming R is String (comments/decidedBy) for the check
        if (value instanceof String) {
            return value != null && !((String) value).trim().isEmpty();
        }
        return value != null;
    }

    // --- Public Getters to retrieve the *actual values* from the entity ---
    
    public R getRetainedField(T entity) {
        return retainedFieldGetter.apply(entity);
    }
    
    public R getDecidedBy(T entity) {
        return decidedByGetter.apply(entity);
    }

    public R getDecision(T entity) {
        return decisionGetter.apply(entity);
    }
}

// Mock Entity structure
public class Entity {
    // Retained Fields
    private final String endorsementComments;
    private final String jobManagerComments;
    private final String regionalManagerComments;
    private final String completionComments;
    
    // DecidedBy Fields
    private final String endorsementDecidedBy;
    private final String jobManagerDecidedBy;
    private final String regionalManagerDecidedBy;
    private final String completionDecidedBy;
    
    // Decision Fields (Note: These are likely reset in your case, but we still need the getter)
    private final String endorsementDecision;
    private final String jobManagerDecision;
    private final String regionalManagerDecision;
    private final String completionDecision;

    // --- Constructor (simplified for example) ---

    // --- Getters for ALL fields ---
    public String getEndorsementComments() { return endorsementComments; }
    public String getJobManagerComments() { return jobManagerComments; }
    public String getRegionalManagerComments() { return regionalManagerComments; }
    public String getCompletionComments() { return completionComments; }

    public String getEndorsementDecidedBy() { return endorsementDecidedBy; }
    public String getJobManagerDecidedBy() { return jobManagerDecidedBy; }
    public String getRegionalManagerDecidedBy() { return regionalManagerDecidedBy; }
    public String getCompletionDecidedBy() { return completionDecidedBy; }

    public String getEndorsementDecision() { return endorsementDecision; }
    public String getJobManagerDecision() { return jobManagerDecision; }
    public String getRegionalManagerDecision() { return regionalManagerDecision; }
    public String getCompletionDecision() { return completionDecision; }
    
    // Example setup for a single scenario
    public Entity(String ec, String edb, String ed, 
                  String jmc, String jmdb, String jmd,
                  String rmc, String rmdb, String rmd,
                  String cc, String cdb, String cd) {
        this.endorsementComments = ec; this.endorsementDecidedBy = edb; this.endorsementDecision = ed;
        this.jobManagerComments = jmc; this.jobManagerDecidedBy = jmdb; this.jobManagerDecision = jmd;
        this.regionalManagerComments = rmc; this.regionalManagerDecidedBy = rmdb; this.regionalManagerDecision = rmd;
        this.completionComments = cc; this.completionDecidedBy = cdb; this.completionDecision = cd;
    }
}

import java.util.Arrays;
import java.util.List;

public class FullyGenericWorkflowService {

    // Define the ordered list of all approval steps.
    private final List<FullyGenericApprovalStep<Entity, String>> approvalSteps;

    public FullyGenericWorkflowService() {
        // Order is crucial: Highest approval level (Completion) down to the lowest (Endorsement)
        this.approvalSteps = Arrays.asList(
            // 1. Completion Approval
            new FullyGenericApprovalStep<>(
                Entity::getCompletionComments, 
                Entity::getCompletionDecidedBy, 
                Entity::getCompletionDecision
            ),
            
            // 2. Regional Manager Approval
            new FullyGenericApprovalStep<>(
                Entity::getRegionalManagerComments, 
                Entity::getRegionalManagerDecidedBy, 
                Entity::getRegionalManagerDecision
            ),
            
            // 3. Job Manager Approval
            new FullyGenericApprovalStep<>(
                Entity::getJobManagerComments, 
                Entity::getJobManagerDecidedBy, 
                Entity::getJobManagerDecision
            ),
            
            // 4. Endorsement
            new FullyGenericApprovalStep<>(
                Entity::getEndorsementComments, 
                Entity::getEndorsementDecidedBy, 
                Entity::getEndorsementDecision
            )
        );
    }
    
    /**
     * Finds the most recently updated approval step by checking the preserved fields
     * in reverse workflow order.
     *
     * @param entity The workflow entity object.
     * @return An Optional containing the FullyGenericApprovalStep that last acted.
     */
    public Optional<FullyGenericApprovalStep<Entity, String>> findLastApprovalStep(Entity entity) {
        
        for (FullyGenericApprovalStep<Entity, String> step : approvalSteps) {
            if (step.hasActed(entity)) {
                return Optional.of(step);
            }
        }
        return Optional.empty();
    }
}


--- Extended with enum and more types not just string

import java.time.LocalDateTime;
import java.util.function.Function;

// T: Entity Type, R: Retained Field Type (e.g., String for Comments)
// D1: DecidedBy/Date Type, D2: Decision Type (e.g., DecisionType Enum)
class FullyGenericApprovalStep<T, R, D1, D2> {
    
    // Getters for the core fields
    private final Function<T, R> retainedFieldGetter; // e.g., Comments (String)
    private final Function<T, D1> decidedByGetter;    // e.g., DecidedBy (String)
    private final Function<T, D2> decisionGetter;     // e.g., Decision (DecisionEnum)
    
    // New Getter for the Decision Date
    private final Function<T, D1> decisionDateGetter; // e.g., DecisionDate (LocalDateTime)

    public FullyGenericApprovalStep(
        Function<T, R> retainedFieldGetter, 
        Function<T, D1> decidedByGetter, 
        Function<T, D2> decisionGetter,
        Function<T, D1> decisionDateGetter) {
        
        this.retainedFieldGetter = retainedFieldGetter;
        this.decidedByGetter = decidedByGetter;
        this.decisionGetter = decisionGetter;
        this.decisionDateGetter = decisionDateGetter;
    }

    /**
     * Checks if the persistent field (Retained Field) is populated.
     */
    public boolean hasActed(T entity) {
        R value = retainedFieldGetter.apply(entity);
        // Retained field (R) is usually String (Comments) or similar reference field
        if (value instanceof String) {
            return value != null && !((String) value).trim().isEmpty();
        }
        return value != null;
    }

    // --- Public Getters to retrieve the *actual values* from the entity ---
    
    public R getRetainedField(T entity) { return retainedFieldGetter.apply(entity); }
    public D1 getDecidedBy(T entity) { return decidedByGetter.apply(entity); }
    public D2 getDecision(T entity) { return decisionGetter.apply(entity); }
    public D1 getDecisionDate(T entity) { return decisionDateGetter.apply(entity); }
}

import java.time.LocalDateTime;

// Example of a custom type (Enum) for the decision field
enum DecisionType {
    ACCEPT, REJECT, WITHDRAWN, PENDING
}

// Updated Entity Mock
public class Entity {
    // Retained Fields (R = String)
    private final String completionComments;
    // DecidedBy Fields (D1 = String)
    private final String completionDecidedBy;
    // Decision Fields (D2 = DecisionType)
    private final DecisionType completionDecision;
    // New Date Field (D1 = LocalDateTime)
    private final LocalDateTime completionDecisionDate;

    // --- Constructor (simplified for one step for brevity) ---
    public Entity(String cc, String cdb, DecisionType cd, LocalDateTime cdd) {
        this.completionComments = cc;
        this.completionDecidedBy = cdb;
        this.completionDecision = cd;
        this.completionDecisionDate = cdd;
    }

    // --- Getters for ALL fields ---
    public String getCompletionComments() { return completionComments; }
    public String getCompletionDecidedBy() { return completionDecidedBy; }
    public DecisionType getCompletionDecision() { return completionDecision; }
    public LocalDateTime getCompletionDecisionDate() { return completionDecisionDate; }
    
    // Add all other step fields (endorsement, jobManager, rsm) here...
    // ...
}

import java.util.Arrays;
import java.util.List;
import java.time.LocalDateTime;

public class FullyGenericWorkflowService {

    // Define the list of approval steps with specific type arguments
    private final List<FullyGenericApprovalStep<Entity, ?, ?, ?>> approvalSteps;

    public FullyGenericWorkflowService() {
        this.approvalSteps = Arrays.asList(
            // 1. Completion Approval Step:
            // T=Entity, R=String (Comments), D1=String (DecidedBy), D2=DecisionType (Decision)
            new FullyGenericApprovalStep<Entity, String, String, DecisionType>(
                Entity::getCompletionComments, 
                Entity::getCompletionDecidedBy, 
                Entity::getCompletionDecision,
                Entity::getCompletionDecisionDate 
            )
            // 2. Add other steps (RSM, JM, Endorsement) similarly
            // ...
        );
    }
    
    // The core function remains the same, but the return type is updated
    public Optional<FullyGenericApprovalStep<Entity, String, String, DecisionType>> findLastApprovalStep(Entity entity) {
        
        // We must cast the result from the generic list to the specific expected type
        for (FullyGenericApprovalStep<Entity, ?, ?, ?> step : approvalSteps) {
            if (step.hasActed(entity)) {
                // Assuming all steps use the same signature for the final result fields
                @SuppressWarnings("unchecked")
                FullyGenericApprovalStep<Entity, String, String, DecisionType> result = 
                    (FullyGenericApprovalStep<Entity, String, String, DecisionType>) step;
                return Optional.of(result);
            }
        }
        return Optional.empty();
    }
}

public class Main {
    public static void main(String[] args) {
        FullyGenericWorkflowService service = new FullyGenericWorkflowService();
        
        // Setup: OP has acted (retention field is populated)
        Entity entityExample = new Entity(
            "Final OP Comment", "OP-Manager", DecisionType.ACCEPT, 
            LocalDateTime.of(2025, 12, 10, 8, 30) // Decision Date
        );
        
        Optional<FullyGenericApprovalStep<Entity, String, String, DecisionType>> result = 
            service.findLastApprovalStep(entityExample);
        
        if (result.isPresent()) {
            FullyGenericApprovalStep<Entity, String, String, DecisionType> lastStep = result.get();
            
            // Retrieving the values (Comments, DecidedBy are String; Decision is Enum; Date is LocalDateTime)
            String lastCommentsValue  = lastStep.getRetainedField(entityExample); // String
            String lastDecidedByValue = lastStep.getDecidedBy(entityExample);     // String
            DecisionType lastDecisionValue = lastStep.getDecision(entityExample);  // DecisionType
            LocalDateTime lastDecisionDateValue = lastStep.getDecisionDate(entityExample); // LocalDateTime

            System.out.println("✅ Last Actor Identified.");
            System.out.println("------------------------------------------");
            System.out.println("Last comments (String):        " + lastCommentsValue);
            System.out.println("Last decidedBy (String):       " + lastDecidedByValue);
            System.out.println("Last decision (DecisionType):  " + lastDecisionValue);
            System.out.println("Last decision date (DateTime): " + lastDecisionDateValue);
        } else {
            System.out.println("No actions recorded.");
        }
    }
}



enum ExpectedActor {
    COMPLETION_APPROVAL,
    REGIONAL_MANAGER_APPROVAL,
    JOB_MANAGER_APPROVAL,
    ENDORSEMENT,
    NONE
}


import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.MethodSource;
import org.junit.jupiter.api.TestInstance;
import org.junit.jupiter.api.Assertions;

import java.time.LocalDateTime;
import java.util.Optional;
import java.util.stream.Stream;

// Note: Ensure the FullyGeneric classes (Entity, ApprovalStep, Service) are available

@TestInstance(TestInstance.Lifecycle.PER_CLASS) // Required for @MethodSource on non-static methods
public class FullyGenericWorkflowServiceTest {

    private final FullyGenericWorkflowService service = new FullyGenericWorkflowService();
    
    // --- Test Data Provider Method ---
    
    // This method returns a stream of arguments, each representing one test case.
    private Stream<Object[]> testCases() {
        // Define common field values for populated steps
        String COMMENT = "C";
        String DECIDED_BY = "D";
        DecisionType DECISION = DecisionType.ACCEPT;
        LocalDateTime DATE = LocalDateTime.now();
        
        // Use null to represent empty/unpopulated fields
        String NULL_S = null;
        DecisionType NULL_D = null;
        LocalDateTime NULL_DT = null;

        return Stream.of(
            // --- Normal Progression Scenarios ---
            
            // 1. DRAFT state (Expected: NONE)
            new Object[]{
                new Entity(NULL_S, NULL_S, NULL_D, NULL_DT, 
                           NULL_S, NULL_S, NULL_D, NULL_DT, 
                           NULL_S, NULL_S, NULL_D, NULL_DT, 
                           NULL_S, NULL_S, NULL_D, NULL_DT), 
                ExpectedActor.NONE
            },
            
            // 2. Awaiting Approval (Last action: Endorsement)
            new Object[]{
                new Entity(COMMENT, DECIDED_BY, DECISION, DATE, // ENDORSEMENT ACTED
                           NULL_S, NULL_S, NULL_D, NULL_DT, 
                           NULL_S, NULL_S, NULL_D, NULL_DT, 
                           NULL_S, NULL_S, NULL_D, NULL_DT), 
                ExpectedActor.ENDORSEMENT
            },
            
            // 3. Awaiting RSM Approval (Last action: Job Manager)
            new Object[]{
                new Entity(COMMENT, DECIDED_BY, DECISION, DATE, 
                           COMMENT, DECIDED_BY, DECISION, DATE, // JM ACTED
                           NULL_S, NULL_S, NULL_D, NULL_DT, 
                           NULL_S, NULL_S, NULL_D, NULL_DT), 
                ExpectedActor.JOB_MANAGER_APPROVAL
            },
            
            // 4. Complete (Last action: Completion)
            new Object[]{
                new Entity(COMMENT, DECIDED_BY, DECISION, DATE, 
                           COMMENT, DECIDED_BY, DECISION, DATE, 
                           COMMENT, DECIDED_BY, DECISION, DATE, 
                           COMMENT, DECIDED_BY, DECISION, DATE), // COMPLETION ACTED
                ExpectedActor.COMPLETION_APPROVAL
            },
            
            // --- Fast-Track Scenarios (Skipping steps) ---
            
            // 5. Fast-Track to Completion (Skipped RSM/Endorsement, Last: Completion)
            new Object[]{
                new Entity(NULL_S, NULL_S, NULL_D, NULL_DT, 
                           COMMENT, DECIDED_BY, DECISION, DATE, // JM ACTED
                           NULL_S, NULL_S, NULL_D, NULL_DT, 
                           COMMENT, DECIDED_BY, DECISION, DATE), // COMPLETION ACTED LAST
                ExpectedActor.COMPLETION_APPROVAL
            },
            
            // 6. Fast-Track to Awaiting RSM Approval (Skipped Endorsement, Last: Job Manager)
            new Object[]{
                new Entity(NULL_S, NULL_S, NULL_D, NULL_DT, 
                           COMMENT, DECIDED_BY, DECISION, DATE, // JM ACTED LAST
                           NULL_S, NULL_S, NULL_D, NULL_DT, 
                           NULL_S, NULL_S, NULL_D, NULL_DT),
                ExpectedActor.JOB_MANAGER_APPROVAL
            },

            // --- Rejection/Rework Scenarios (Intermediate actor is the last) ---

            // 7. Awaiting Revision (Rejected by OP, all fields are populated)
            // The highest populated field is Completion, which is the last action.
            new Object[]{
                new Entity(COMMENT, DECIDED_BY, DECISION, DATE, 
                           COMMENT, DECIDED_BY, DECISION, DATE, 
                           COMMENT, DECIDED_BY, DECISION, DATE, 
                           COMMENT, DECIDED_BY, DECISION, DATE), // COMPLETION ACTED LAST
                ExpectedActor.COMPLETION_APPROVAL
            },
            
            // 8. Awaiting Approval (Rejected by RSM, Completion fields are NULL)
            // The highest populated field is RSM, which is the last action.
            new Object[]{
                new Entity(COMMENT, DECIDED_BY, DECISION, DATE, 
                           COMMENT, DECIDED_BY, DECISION, DATE, 
                           COMMENT, DECIDED_BY, DECISION, DATE, // RSM ACTED LAST
                           NULL_S, NULL_S, NULL_D, NULL_DT),
                ExpectedActor.REGIONAL_MANAGER_APPROVAL
            }
        );
    }

    // --- The Parameterized Test Method ---

    @ParameterizedTest
    @MethodSource("testCases")
    void testFindLastApprovalStep_WithVariousScenarios(Entity entity, ExpectedActor expectedActor) {
        
        Optional<FullyGenericApprovalStep<Entity, String, String, DecisionType>> result = 
            service.findLastApprovalStep(entity);

        if (expectedActor == ExpectedActor.NONE) {
            // Case 1: Expected no actor (e.g., Draft)
            Assertions.assertTrue(result.isEmpty(), "Expected no actor to have acted, but found one.");
            return;
        }

        Assertions.assertTrue(result.isPresent(), "Expected an actor to have acted, but none was found.");
        
        // 1. Get the actual actor's comments field value (the retained field)
        String lastCommentValue = result.get().getRetainedField(entity);
        
        // 2. Check which expected field the result corresponds to
        switch (expectedActor) {
            case COMPLETION_APPROVAL:
                Assertions.assertEquals(entity.getCompletionComments(), lastCommentValue, "Expected Completion Approval");
                break;
            case REGIONAL_MANAGER_APPROVAL:
                Assertions.assertEquals(entity.getRegionalManagerComments(), lastCommentValue, "Expected RSM Approval");
                break;
            case JOB_MANAGER_APPROVAL:
                Assertions.assertEquals(entity.getJobManagerComments(), lastCommentValue, "Expected Job Manager Approval");
                break;
            case ENDORSEMENT:
                Assertions.assertEquals(entity.getEndorsementComments(), lastCommentValue, "Expected Endorsement");
                break;
            default:
                Assertions.fail("Unexpected ExpectedActor value: " + expectedActor);
        }
        
        // Add additional assertions here if needed, e.g., checking the DECIDED_BY value.
        // String lastDecidedByValue = result.get().getDecidedBy(entity);
        // Assertions.assertEquals(DECIDED_BY, lastDecidedByValue); 
    }
}
