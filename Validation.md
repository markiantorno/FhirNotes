**BaseHapiFhirResourceDao** houses the validator which we use to validate **Resource**.

The validator is an instance of `ca.uhn.fhir.FhirValidator`.
We get the instance of this validator through calling `getContext().newValidator()`

`getContext()` is a public method that returns an instance of `ca.uhn.fhir.FhirContext`. 

_The FHIR context is the central starting point for the use of the HAPI FHIR API. It should be created once, and then 
used as a factory for various other types of objects (parsers, clients, etc.)._

By calling `newValidator()` on the **FhirContext**, we get a new instance of a `ca.uhn.fhir.validation.FhirValidator`

_The FhirValidator is a resource validator, which checks resources for compliance against various validation schemes 
(schemas, schematrons, profiles, etc.)_

From **BaseHapiFhirResourceDao**, we then call `validateWithResult(theRawResource, options)`, where theResource is a 
String representation of the **Resource** to validate, and options is an instance of `ca.uhn.fhir.validation.ValidationOptions`.

The **ValidationOptions** is just created in the **BaseHapiFhirResourceDao** using the base constructor, 
`ValidationOptions options = new ValidationOptions().addProfileIfNotBlank(theProfile);`

The `profile` argument we pass in is a String URI specifying a profile to use. In our case for tests, this is `null`.

The output from this method is `ca.uhn.fhir.validation.ValidationResult` object, which contains the **FhirContext**, a 
`boolean myIsSuccessful`, and a `List` of messages.

Within the method `FhirValidator.validateWithResult(...)` we:

1. Validate the **Resource** data passed in is not null
1. Apply the default validators`
    * In the class there is a list of `ca.uhn.fhir.validation.IValidatorModule`
    * An individual validation module, which applies validation rules against resources and adds failure/informational messages as it goes._
    * This is just an interface with the method `void validateResource(IValidationContext<IBaseResource> theCtx);`
    * Validators are added to the **FhirValidator** through the method `registerValidatorModule(IValidatorModule theValidator)`
1. Sets the **FhirValidator** to validate against the standard schema or not 
1. Sets something to do with `validateAgainstStandardSchematron(boolean)` ***JAMES*** What is a schematron... `FhirValidator.setValidateAgainstStandardSchematron(...)`
1. Converts the passed in **String** resource to a `ca.uhn.fhir.validation.IValidationContext`
    * This is just a class that stores the **FhirContext**, **Resource**, and various fields for the validation results
1. We then iterate through all the **IValidatorModule** in the class and call the method `validateResource(IValidationContext<IBaseResource> ctx)` passing in our newly **IValidationContext**
1. We then return the **ValidationResult**

So the instance of IValidatorModule in and instance of `org.hl7.fhir.r4.hapi.validation.FhirInstanceValidator`, which 
extends `org.hl7.fhir.r4.hapi.validation.BaseValidatorBridge`

***JAMES*** What's up with the **BaseValidatorBridge**? Why do we need a base class for a bridge between the RI(?) validation tools and HAPI?

The **FhirInstanceValidator** doesn't actually override the `doValidate(...)` method in **BaseValidatorBridge**, it does 
however, hold an instance of `FhirInstanceValidator.WorkerContextWrapper`

So, the **BaseInstanceValidator** calls the abstract method `validate(IValidationContext<?> theValidationContext`, which is 
implemented in `org.hl7.fhir.r4.hapi.validation.FhirInstanceValidator`

In this `validate(...)` method, we:
1. Get the instance of the `FhirInstanceValidator.WorkerContextWrapper`
    * If the instance is null, we initialize by passing in the **FhirContext** and the instance of **IValidationSupport**, this is an instance of `org.hl7.fhir.r4.hapi.validation.CachingValidationSupport`
    * This is an interesting point here. The **CachingValidationSupport** class maintains an instance of a **Caffeine** cache, this cache stores results from calls done on the **IValidationSupport**
1. Set all the preferences for the **ValidationWrapper** 
1. Call `validate(wrapperWorkerContext, theValidationCtx)` on the ValidatorWrapper
1. Return the result of this call

In the `org.hl7.fhir.common.hapi.validation.ValidationWrapper` class, we create an instance of an `r5` **InstanceValidator** by passing the **IWorkerContect** and **IValidationContext** to it's constructor. 
Then, the **ValidatorWrapper** determines if the encoding is XML, JSON, or other, parses the resource, and then calls `InstanceValidator.validate(..)`, passing in:
* The `appContext` as an **Object** ***JAMES*** Why is this passed as an object? HAPI passes this as null, so I'm assuming it's not needed here? It is eventually passed to a **ValidatorHostContext**, which is `r5`...so is this a choice, or just a holdover from there being 5 different validators?
* A `List<ValidationMessage>` that will hold the results of the validation
* The `InputStream` of data for the parsed resource
* The determined `Manager.FhirFormat`, ie `Manager.FhirFormat.JSON`
* A list of profiles we loaded in this method, if present. A validation operation doesn't necessarily need to have profiles, it can validate against the default

In the **InstanceValidator**, we:
1. Create an instance of `org.hl7.fhir.r5.elementmodel.ParserBase` passing in our
    * **WorkingContextWrapper**
    * **FhirFormat**
1. Pass the **ParserBase** our list of **ValidationMessages*** to populate, and out **ValidationPolicy**, which defaults to `ValidationPolicy.EVERYTHING`
1. Attempt to parse the **InputStream** for the resource
1. If an error is not thrown by the parser, we finish. The `List<ValidationMessages` is already populated from our call to `parser.parse(...)` because we passed it to the parser before attempting. 
1. If the parse was successful, we then call `InstanceValidator.validate` passing in
    * The **IValidationContext** ...which is still being passed as an object, I have no idea why
    * The `List<ValidationMessages>`
    * The successfully parsed resource, as an `org.hl7.fhir.r5.elementmodel.Element`
    * The list of profiles we loaded earlier, if any exist

From here, this is the main entry point for validation. All all the other public entry points end up here coming here.
The first thing to do is to clear the internal state.
* We have a fetchCatch which is a `Map<String, Element>` that stores the `Element` mapped to it's `element.fhirType()+"/"+element.getIdBase()`
* A resourceTracker, as a `HashMap<Element, ResourceValidationTracker>`, which maps each `Element` to a `InstanceValidator.ResourceValidationTracker`
    * The validator keeps a **ResourceValidationTracker** for each resource it validates. It keeps track of the state of resource validation - a given resource may be validated against multiple profiles during a single validation, and it may be valid against some, and invalid against others. We don't want to keep doing the same validation, so we remember the outcomes for each combination of resource + profile
* An executionId, which is sumply a generated **UUID**, unique to this validation
* baseOnly, a **Boolean** set to true if no valid profiles were passed in. True == We are validating against the base fhir spec profile only

After all those have been reset for this run, we call `InstanceValidator.validateResource(...)`, which is the actual base entry point for internal use.
This method call takes:
* `org.hl7.fhir.r5.validation.ValidatorHostContext` which we create a new instance of inline when we call the constructor
    * This takes in both a `InstanceValidator.ValidatorHostContext` (The `appContext` we've been passing around as an **Object**), and `org.hl7.fhir.r5.model.StructureDefinition` (the successfully parsed **Element**)
* The `List<ValidationMessages>`, to populate
* The successfully parsed **Element**
* The successfully parser ** Element**  ***JAMES*** Why is this a thing? We pass in the parsed Element twice? Is there ever a case where these would be different?
* The **StructureDefinition** we are validating against, or null if we're validating against the default
* Our `org.hl7.fhir.r5.utils.IdStatus`, which will be one of `OPTIONAL`, `REQUIRED`, or `PROHIBITED` (...just defines what the rule around resource ID is)
* A new instance of a `org.hl7.fhir.r5.validation.NodeStack` that is created inline with the parsed `Element` as an argument

_The same call to `InstanceValidator.validateResource` is made regardless of if there are any **StructureDefinitions** or not. The only difference is that in the case where there are none passes in, we pass null as the argument, otherwise we iterate through the list of **StructureDefinitions** and call `InstanceValidator.validateResource` for each._

In `validateResource(...)` we:
1. Check if there is a valid **StructureDefinition** passed in
    * If not, we get the **ResourceName** from the **Element** and use it to fetch the default **StructureDefinition** for that type from the current **FhirContext**`context.fetchResource(StructureDefinition.class, "http://hl7.org/fhir/StructureDefinition/" + resourceName);`
    * If no such **StructureDefinition** is found for the given resource name, we make note of this in a **Boolean** variable
1. If the **Element** is a **Bundle**, we take the first entry in the **Bundle** and validate against it
    * _validating against multiple entries within the same **Bundle** is not currently supported_
1. If no such **StructureDefinition** was found in the earlier step, or the passed in **StructureDefionition** is null, we generate an error message via the `InstanceValidator.rule(...)` method
1. Otherwise, if the **StructureDefinition** is valid
    1. Check if the **IdStatus** is `REQUIRED` and the id of the **Element** is not set
    2. Check if the **IdStatus** is `PROHIBITED` and the id of the **Element** is set
    3. Add error messages based on the result of those two operations
1. Call `InstanceValidator.start(...)`








**N.B.** The best way to debug validation errors is to go to `org.hl7.fhir.utilities.validation.ValidationMessage` and put a breakpoint in each of the constructors in that class. It is essentially the main entry point for any error or warning creation in the validation process, and you can trace back the issue from there.






