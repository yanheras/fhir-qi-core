{:toc}

{: #QICore-Patterns}

The patterns described here have been developed through usage of QI-Core profiles in the development of
CQL-based quality measures and decision support. CMS program measures can be accessed at [HealthIT.gov](https://ecqi.healthit.gov/). Multiple repositories of draft FHIR-based measures have been developed and are indexed in the [eCQM Content Index](https://github.com/cqframework/ecqm-content) repository.

> NOTE: The examples in this section follow recommended best-practices as of the time of publication. However, these practices are constantly being revised based on implementer and community feedback. For a complete discussion of authoring patterns, see the [Authoring Patterns](https://github.com/cqframework/CQL-Formatting-and-Usage-Wiki/wiki/Authoring-Patterns---QICore-v5.0.0) topic in the CQL Formatting and Usage Wiki.

>NOTE: Patterns page for 4.1.1 has been superseded by [patterns wiki for 4.1.1](https://github.com/cqframework/CQL-Formatting-and-Usage-Wiki/wiki/Authoring-Patterns---QICore-v4.1.1)

### FHIR and CQL

Clinical Quality Language ([CQL](http://cql.hl7.org)) is a high-level, domain-specific language focused on clinical quality and targeted at clinical knowledge artifact authors such as quality measure and decision support artifact developers. In addition, the CQL specification provides a machine-readable canonical representation called Expression Logical Model ([ELM](https://cql.hl7.org/04-logicalspecification.html)) targeted at implementations and designed to enable automated sharing of clinical knowledge.

To use CQL with FHIR, [model information](https://cql.hl7.org/07-physicalrepresentation.html#data-model-references) must be provided to the implementation environment. The [CQF Common](http://fhir.org/guides/cqf/common) IG provides a FHIR-ModelInfo library that provides this information for the base FHIR specification, as well as FHIRHelpers and FHIRCommon libraries that provide commonly used functions and declarations for clinical knowledge artifact development. To use FHIR directly, follow the documentation provided in that implementation guide.

However, this implementation guide includes a [QICore ModelInfo](Library-QICore-ModelInfo.html) library that provides model information for the profiles and extensions defined in QI-Core. To use this model, include a [using declaration](https://cql.hl7.org/02-authorsguide.html#data-models) as shown in the example below:

```cql
using QICore version '5.0.0'
```

Although not required by CQL, current best-practice is to include the version of the QICore model. For more information about how this library is constructed, refer to the [ModelInfo](modelinfo.html) topic.


#### Primitives

The QICore model info represents FHIR primitive elements using the system types directly, so when using QICore, no `.value` accessor is required.

To facilitate implementation in FHIR, references to primitive elements are translated to the FHIR representation using the same `FHIRHelpers` library used for pure-FHIR artifacts, so this library is still required as a runtime, rather than a compile-time dependency for profile-informed authoring models.

In profile-informed authoring, the primitive type is mapped to the CQL type and represented with the CQL type in the model. So a `FHIR.string` element is modeled as a `System.String`, and the translator maps that to the `FHIR.string` value using the `FHIRHelpers` library.

```cql
include FHIRHelpers version '4.0.1'
```

Note that the FHIRHelpers version must be consistent with the FHIR version being used.

#### Extensions

Extensions in FHIR provide a standard mechanism to describe additional content that is not part of the base
FHIR resources. By defining extensions in a uniform way as part of the base specification, FHIR enables extension-based functionality to be introduced through the use of profiles and implementation guidance. QI-Core includes several extensions related to quality improvement applications.

With profile-informed authoring in QICore, extensions and slices defined in profiles are represented directly as elements in the types. For example:

```cql
define TestSlices:
  [USCoreBloodPressure] BP
    where BP.systolic.value < 140 'mm[Hg]'
      and BP.diastolic.value < 90 'mm[Hg]'

define TestSimpleExtensions:
  Patient P
    where P.birthsex = 'M'

define TestComplexExtensions:
  Patient P
    where P.race.ombCategory contains "American Indian or Alaska Native"
      and P.race.detailed contains "Alaska Native"
```

#### Choice Types

FHIR includes the notion of Choice Types, or elements that can be of any of a number of types. For example,
the `Patient.deceased` element can be specified as a `Boolean` or as a `DateTime`. Since CQL also supports choice types, these are manifest directly as Choice Types within the Model Info.

Where appropriate, QICore profiles restrict choice types to those that are appropriate for the quality improvement use case. For example, The `QICoreConditionProblemsHealthConcerns` profile removes `String` as a possible type for the `onset` element, to communicate the expectation that a computable representation of onset is required for quality improvement applications.

Quality improvement logic must be prepared to deal with choice elements of any of the available types because systems may communicate instances with values of any of these types. To support the most common usages of choice types (for timing elements), the following functions have been defined:

```cql
define fluent function toInterval(choice Choice<DateTime, Quantity, Interval<DateTime>, Interval<Quantity>>):
  case
	  when choice is DateTime then
    	Interval[choice as DateTime, choice as DateTime]
		when choice is Interval<DateTime> then
  		choice as Interval<DateTime>
		when choice is Quantity then
		  Interval[Patient.birthDate + (choice as Quantity),
			  Patient.birthDate + (choice as Quantity) + 1 year)
		when choice is Interval<Quantity> then
		  Interval[Patient.birthDate + (choice.low as Quantity),
			  Patient.birthDate + (choice.high as Quantity) + 1 year)
		else
			null as Interval<DateTime>
	end

define fluent function abatementInterval(condition Condition):
	if condition.abatement is DateTime then
	  Interval[condition.abatement as DateTime, condition.abatement as DateTime]
	else if condition.abatement is Quantity then
		Interval[Patient.birthDate + (condition.abatement as Quantity),
			Patient.birthDate + (condition.abatement as Quantity) + 1 year)
	else if condition.abatement is Interval<Quantity> then
	  Interval[Patient.birthDate + (condition.abatement.low as Quantity),
		  Patient.birthDate + (condition.abatement.high as Quantity) + 1 year)
	else if condition.abatement is Interval<DateTime> then
	  Interval[condition.abatement.low, condition.abatement.high)
	else null as Interval<DateTime>

define fluent function prevalenceInterval(condition Condition):
if condition.clinicalStatus ~ "active"
  or condition.clinicalStatus ~ "recurrence"
  or condition.clinicalStatus ~ "relapse" then
  Interval[start of ToInterval(condition.onset), end of ToAbatementInterval(condition)]
else
  Interval[start of ToInterval(condition.onset), end of ToAbatementInterval(condition))

define fluent function latest(choice Choice<DateTime, Quantity, Interval<DateTime>, Interval<Quantity>> ):
  (choice.toInterval()) period
    return
      if period.hasEnd() then end of period
      else start of period

define fluent function earliest(choice Choice<DateTime, Quantity, Interval<DateTime>, Interval<Quantity>> ):
  (choice.toInterval()) period
    return
      if period.hasStart() then start of period
      else end of period
```

Note that these functions make use of the FHIRHelpers library to ensure correct processing.

> NOTE: The examples throughout this topic have been simplified to illustrate specific usage. Refer to the originating context for complete expressions.
