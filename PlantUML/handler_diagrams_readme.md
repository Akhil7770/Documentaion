# Handler Chain PlantUML Diagrams

This directory contains comprehensive PlantUML diagrams explaining the **Handler Chain** architecture of the Cost Estimator Calculation Service.

## 📊 Diagram Files

### 1. **handler_flow_diagram.puml** - Complete Activity Flow
- **Type:** Activity Diagram
- **Purpose:** Shows the complete flow through all 10 handlers with detailed decision points
- **Best For:** Understanding the full calculation logic step-by-step
- **Complexity:** High (most detailed)

**What it shows:**
- All 10 handlers in sequence
- Every decision point (if/else conditions)
- Calculations performed at each step
- Terminal conditions
- Common paths through the chain

---

### 2. **handler_sequence_diagram.puml** - Sequential Flow
- **Type:** Sequence Diagram
- **Purpose:** Shows how handlers call each other in sequence
- **Best For:** Understanding handler interactions and message passing
- **Complexity:** Medium

**What it shows:**
- Handler-to-handler communication
- Multiple scenario flows:
  - Deductible not met
  - Deductible already met
  - OOPMax already met
  - Service not covered
- Actual calculation values in examples

---

### 3. **handler_state_diagram.puml** - State Transitions
- **Type:** State Diagram
- **Purpose:** Shows all possible state transitions between handlers
- **Best For:** Understanding routing possibilities and handler relationships
- **Complexity:** Medium

**What it shows:**
- All handlers as states
- All possible transitions (arrows)
- Conditions for each transition
- Terminal states
- Handler types (Routing/Calculation/Terminal)

---

### 4. **handler_class_diagram.puml** - Class Structure
- **Type:** Class Diagram
- **Purpose:** Shows the object-oriented structure and dependencies
- **Best For:** Understanding code architecture and handler relationships
- **Complexity:** Low (structural overview)

**What it shows:**
- Handler inheritance from base Handler class
- InsuranceContext data structure
- Handler dependencies (which handlers reference which)
- Public and private methods
- CalculationServiceImpl's role

---

## 🚀 How to View These Diagrams

### Option 1: Online PlantUML Viewer (Easiest)
1. Go to: https://www.plantuml.com/plantuml/uml/
2. Copy the contents of any `.puml` file
3. Paste into the text area
4. Click "Submit" to render

### Option 2: VS Code Extension
1. Install "PlantUML" extension in VS Code
2. Open any `.puml` file
3. Press `Alt+D` (or right-click → "Preview Current Diagram")
4. View rendered diagram in split pane

### Option 3: IntelliJ IDEA Plugin
1. Install "PlantUML integration" plugin
2. Open any `.puml` file
3. Diagram renders automatically in side panel

### Option 4: Command Line (requires Graphviz)
```bash
# Install PlantUML and Graphviz first
brew install graphviz plantuml  # macOS
# or
apt-get install graphviz plantuml  # Linux

# Generate PNG images
plantuml handler_flow_diagram.puml
plantuml handler_sequence_diagram.puml
plantuml handler_state_diagram.puml
plantuml handler_class_diagram.puml

# This creates PNG files in the same directory
```

---

## 📚 Understanding the Handler Chain

### The 10 Handlers (in order)

1. **ServiceCoverageHandler** - Is service covered?
2. **BenefitLimitationHandler** - Check annual limits
3. **OOPMaxHandler** - Is OOPMax met? (Routing)
4. **OOPMaxCopayHandler** - Apply copay when OOPMax met (Terminal)
5. **DeductibleHandler** - Check deductible status (Routing)
6. **CostShareCoPayHandler** - Apply copay (no deductible)
7. **DeductibleCostShareCoPayHandler** - Route after deductible met
8. **DeductibleCoPayHandler** - Apply copay with deductible (Complex!)
9. **DeductibleOOPMaxHandler** - Apply deductible payment
10. **DeductibleCoInsuranceHandler** - Apply coinsurance (Terminal)

### Handler Types

| Type | Handlers | Description |
|------|----------|-------------|
| **Routing** | #3, #5, #7 | Make decisions, route to next handler |
| **Calculation** | #6, #8, #9 | Calculate amounts AND route |
| **Terminal** | #4, #10 | Can end calculation |
| **Hybrid** | #1, #2 | Can route OR terminate |

### Common Paths Through the Chain

**Path 1: Normal Flow (Deductible Not Met)**
```
1. ServiceCoverage (covered ✓)
   ↓
2. BenefitLimitation (no limits ✓)
   ↓
3. OOPMax (not met ✓)
   ↓
5. Deductible (not met)
   ↓
9. DeductibleOOPMax (apply $500 deductible)
   ↓
7. DeductibleCostShareCoPay (deductible now met)
   ↓
10. DeductibleCoInsurance (apply 20% coinsurance)
    ↓
    DONE ✓
```

**Path 2: Deductible Already Met**
```
1. ServiceCoverage (covered ✓)
   ↓
2. BenefitLimitation (no limits ✓)
   ↓
3. OOPMax (not met ✓)
   ↓
5. Deductible (already met ✓)
   ↓
7. DeductibleCostShareCoPay
   ↓
8. DeductibleCoPay (apply $30 copay)
   ↓
10. DeductibleCoInsurance (apply 20% coinsurance)
    ↓
    DONE ✓
```

**Path 3: OOPMax Already Met**
```
1. ServiceCoverage (covered ✓)
   ↓
2. BenefitLimitation (no limits ✓)
   ↓
3. OOPMax (MET! ✓)
   ↓
4. OOPMaxCopay (member pays $0)
   ↓
   DONE ✓ (Insurance pays 100%)
```

**Path 4: Service Not Covered**
```
1. ServiceCoverage (NOT covered ❌)
   ↓
   DONE ❌ (Member pays 100%)
```

---

## 🎯 Key Concepts

### InsuranceContext
The central data container that flows through all handlers. Contains:
- **Input:** service_amount
- **Rules:** is_deductible_before_copay, copay_applies_oop, etc.
- **Accumulators:** deductible_calculated, oopmax_calculated, etc.
- **Output:** member_pays, calculation_complete
- **Tracing:** trace_entries for debugging

### Chain of Responsibility Pattern
- Each handler can process the request OR pass to next handler
- Handlers are linked together in a chain
- Request flows through chain until `calculation_complete = TRUE`
- Dynamic routing: next handler depends on context values

### Handler Methods
- **process():** Contains business logic (abstract, must implement)
- **handle():** Orchestrates flow (calls process, then routes to next)
- **set_next():** Links handlers together in chain
- **set_*_handler():** Sets alternative routing paths (for branching)

---

## 💡 Use Cases for Each Diagram

### When to use **handler_flow_diagram.puml**:
- ✅ Learning the complete calculation logic
- ✅ Understanding all decision points
- ✅ Debugging calculation issues
- ✅ Documenting business rules

### When to use **handler_sequence_diagram.puml**:
- ✅ Understanding handler interactions
- ✅ Seeing real examples with values
- ✅ Explaining to stakeholders
- ✅ Testing scenario flows

### When to use **handler_state_diagram.puml**:
- ✅ Understanding all possible paths
- ✅ Identifying routing logic
- ✅ Finding handler dependencies
- ✅ Planning new handler additions

### When to use **handler_class_diagram.puml**:
- ✅ Understanding code structure
- ✅ Refactoring handlers
- ✅ Onboarding new developers
- ✅ Planning architectural changes

---

## 🔍 Tips for Understanding

1. **Start with the sequence diagram** - easiest to understand
2. **Then review the state diagram** - see all paths
3. **Study the flow diagram** - detailed logic
4. **Reference the class diagram** - code structure

---

## 📝 Notes

- All diagrams are **synchronized** with the actual code (as of Oct 2025)
- Diagrams show **dynamic routing** - next handler depends on context values
- **Handler #8 (DeductibleCoPayHandler)** is the most complex (271 lines)
- **Handler #10 (DeductibleCoInsuranceHandler)** is always the last handler
- **InsuranceContext** flows through all handlers (modified by each)

---

## 🤝 Contributing

When modifying handlers, please update the corresponding PlantUML diagrams:
1. Identify which diagram(s) are affected
2. Update the `.puml` file
3. Test rendering
4. Commit with handler code changes

---

## 📖 Additional Resources

- **Handler Code:** `/app/services/handlers/*.py`
- **Base Handler:** `/app/core/base.py` (Handler class)
- **Calculation Service:** `/app/services/impl/calculation_service_impl.py`
- **Insurance Context:** `/app/core/base.py` (InsuranceContext dataclass)

---

**Generated:** October 2025  
**Status:** ✅ Up-to-date with codebase  
**Maintainer:** Cost Estimator Team
