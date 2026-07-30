## small/CustomDatatypeUtilTest.java

- **Package:** `org.openmrs.customdatatype`
- **Type:** JUnit 5 test class, extends `BaseContextSensitiveTest`
- **Imports needed:** `java.util.HashMap`, `java.util.Map`, `org.junit.jupiter.api.Test`, `org.openmrs.test.jupiter.BaseContextSensitiveTest`, static `org.junit.jupiter.api.Assertions.assertEquals`
- **Purpose:** Verifies that `CustomDatatypeUtil.deserializeSimpleConfiguration(String)` correctly reverses `CustomDatatypeUtil.serializeSimpleConfiguration(Map)`.
- **Method:** `deserializeSimpleConfiguration_shouldDeserializeAConfigurationSerializedByTheCorrespondingSerializeMethod()` — no return value, annotated `@Test`.
  - Build a `Map<String, String>` with two entries: `"one property" -> "one value"` and `"another property" -> "another value < with > strange&nbsp;characters"` (the second value deliberately contains characters that need escaping, e.g. `<`, `>`, `&`).
  - Call `CustomDatatypeUtil.serializeSimpleConfiguration(config)` to get a serialized `String`.
  - Call `CustomDatatypeUtil.deserializeSimpleConfiguration(serialized)` to get back a `Map<String, String>`.
  - Assert the deserialized map has size 2, and that both keys map back to their original (unescaped) values exactly.
- **Naming convention:** OpenMRS test methods follow `methodUnderTest_shouldExpectedBehavior` naming.

---

## small/LocationEditor.java

- **Package:** `org.openmrs.propertyeditor`
- **Type:** Public class `LocationEditor`, extends a generic base class `OpenmrsPropertyEditor<Location>`
- **Imports needed:** `org.openmrs.Location`, `org.openmrs.api.context.Context`
- **Purpose:** A Spring `PropertyEditor`-style class that lets Spring convert between a `Location` domain object and a plain string (its ID or UUID) so HTML forms can bind to it.
- **Class javadoc:** Explains it serializes/deserializes an object to a string for form binding, and notes that since version 1.9 it can also resolve objects by UUID.
- **Constructor:** `public LocationEditor()` — empty body.
- **Method `getObjectById(Integer id)`:** protected, `@Override`, returns `Location`. Body: `return Context.getLocationService().getLocation(id);`
- **Method `getObjectByUuid(String uuid)`:** protected, `@Override`, returns `Location`. Body: `return Context.getLocationService().getLocationByUuid(uuid);`
- All the shared logic (parsing the string, deciding whether it's numeric ID vs UUID, `setAsText`/`getAsText`) lives in the abstract parent `OpenmrsPropertyEditor<T>`, which this subclass does not need to touch — it only supplies the two lookup hooks.

---

## small/OpenmrsCharacterEscapes.java

- **Package:** `org.openmrs` (root package)
- **Type:** Public class `OpenmrsCharacterEscapes`, extends Jackson's `CharacterEscapes` (`com.fasterxml.jackson.core.io.CharacterEscapes`)
- **Imports needed:** `com.fasterxml.jackson.core.SerializableString`, `com.fasterxml.jackson.core.io.CharacterEscapes`, `com.fasterxml.jackson.core.io.SerializedString`
- **Purpose:** Passed to a Jackson `ObjectMapper` so that when objects are serialized to JSON, HTML-significant characters (`<` and `>`) are escaped, preventing embedded HTML/script injection in JSON output.
- **Field:** `private final int[] asciiEscapes;`
- **Constructor:** 
  - Start from `CharacterEscapes.standardAsciiEscapesForJSON()` to get the baseline escape table.
  - Force custom escaping for `<` and `>` by setting `esc['<'] = CharacterEscapes.ESCAPE_CUSTOM;` and `esc['>'] = CharacterEscapes.ESCAPE_CUSTOM;`.
  - Assign the result to `asciiEscapes`.
- **Method `getEscapeCodesForAscii()`:** `@Override`, returns `int[]`, simply returns `asciiEscapes`.
- **Method `getEscapeSequence(int ch)`:** `@Override`, returns `SerializableString`. If `ch == '<'`, return `new SerializedString("&lt;")`. Else if `ch == '>'`, return `new SerializedString("&gt;")`. Otherwise return `null`.

---

## small/OpenmrsThreadPoolHolder.java

- **Package:** `org.openmrs.util`
- **Type:** Public class `OpenmrsThreadPoolHolder`
- **Imports needed:** `java.util.concurrent.ExecutorService`, `java.util.concurrent.Executors`
- **Purpose:** A minimal holder exposing a single shared, application-wide thread pool.
- **Content:** Exactly one field — `public static final ExecutorService threadExecutor = Executors.newCachedThreadPool();` — no constructor, no methods. It is a static-holder utility class used elsewhere in the codebase to submit background tasks without each caller creating its own executor.

---

## small/PatientSaveHandler.java

- **Package:** `org.openmrs.api.handler`
- **Type:** Public class `PatientSaveHandler`, implements `SaveHandler<Patient>`
- **Annotation:** `@Handler(supports = Patient.class)` — registers this class in OpenMRS's AOP-based handler framework so it's auto-invoked whenever a `Patient` is saved.
- **Imports needed:** `java.util.Date`, `org.openmrs.Patient`, `org.openmrs.PatientIdentifier`, `org.openmrs.User`, `org.openmrs.annotation.Handler`, `org.openmrs.aop.RequiredDataAdvice`
- **Purpose:** Ensures data integrity on `Patient` objects at save time — specifically, that each `PatientIdentifier` attached to a patient has its back-reference to that patient set.
- **Method `handle(Patient patient, User creator, Date dateCreated, String other)`:** `@Override`, `void`.
  - If `patient.getIdentifiers()` is not null, loop over each `PatientIdentifier pIdentifier`.
  - For each identifier, if `pIdentifier.getPatient() == null`, call `pIdentifier.setPatient(patient)` to link it back to the owning patient.
  - No other side effects; `creator`, `dateCreated`, and `other` parameters are part of the interface signature but unused in this implementation.

---

## medium/ConnectionPoolConfigTest.java

- **Package:** `org.openmrs.api.db.hibernate`
- **Type:** JUnit 5 test class `ConnectionPoolConfigTest`, extends `BaseContextSensitiveTest`
- **Imports needed:** `java.util.stream.Stream`, `javax.sql.DataSource`, `org.hibernate.SessionFactory`, `org.hibernate.engine.jdbc.connections.spi.ConnectionProvider`, `org.hibernate.engine.spi.SessionFactoryImplementor`, `org.junit.jupiter.api.Test`, `org.openmrs.test.jupiter.BaseContextSensitiveTest`, `org.springframework.beans.factory.annotation.Autowired`, `com.mchange.v2.c3p0.PoolBackedDataSource`, `com.mchange.v2.c3p0.WrapperConnectionPoolDataSource`, static `assertEquals`, static `assumeTrue`
- **Purpose:** Regression test verifying that the c3p0 connection-pool settings declared in `hibernate.default.properties` (checkout timeout and max pool size) actually take effect on the live pool, guarding against silent configuration-loading regressions that could cause the app to hang under load.
- **Field:** `@Autowired private SessionFactory sessionFactory;`
- **Method `shouldApplyConfiguredConnectionPoolLimits()`:** `@Test`, `void`.
  - Use `assumeTrue(...)` to skip the test if a local `runtime.properties` file overrides any of the c3p0 pool keys (`hibernate.c3p0.checkoutTimeout`, `c3p0.checkoutTimeout`, `hibernate.c3p0.max_size`, `c3p0.max_size`) — check this via `Stream.of(...).noneMatch(runtimeProperties::containsKey)`, referencing a `runtimeProperties` map available from the base test class.
  - Cast `sessionFactory` to `SessionFactoryImplementor`, get its service registry, and fetch the `ConnectionProvider` service.
  - Unwrap the `ConnectionProvider` to a `DataSource`, cast it to `PoolBackedDataSource`, then get its underlying `WrapperConnectionPoolDataSource`.
  - Assert `pool.getCheckoutTimeout() == 30000` and `pool.getMaxPoolSize() == 50`.
- **Class javadoc:** Explain that this pins the default checkout timeout (30s) and max pool size (50), and that other tests (e.g. an order-number-generation concurrency test) derive their thread counts from the max pool size, so changing this value has ripple effects.

---

## medium/HL7SourceValidatorTest.java

- **Package:** `org.openmrs.validator`
- **Type:** JUnit 5 test class `HL7SourceValidatorTest`, extends `BaseContextSensitiveTest`
- **Imports needed:** `org.junit.jupiter.api.Test`, `org.openmrs.hl7.HL7Source`, `org.openmrs.test.jupiter.BaseContextSensitiveTest`, `org.springframework.validation.BindException`, `org.springframework.validation.Errors`, static `assertFalse`, static `assertTrue`
- **Purpose:** Tests the `HL7SourceValidator.validate(Object, Errors)` method, focused on field-length validation for the `name` property of `HL7Source`.
- **Method `validate_shouldPassValidationIfFieldLengthsAreCorrect()`:** `@Test`, `void`.
  - Create `HL7Source hl7Source = new HL7Source()`, call `hl7Source.setName("name")`.
  - Create `Errors errors = new BindException(hl7Source, "hl7Source")`.
  - Call `new HL7SourceValidator().validate(hl7Source, errors)`.
  - Assert `assertFalse(errors.hasErrors())`.
- **Method `validate_shouldFailValidationIfFieldLengthsAreNotCorrect()`:** `@Test`, `void`.
  - Same setup, but set `name` to a very long repeated string (well over the field's max length, e.g. the phrase "too long text " repeated many times, totaling several hundred characters).
  - Validate, then assert `assertTrue(errors.hasFieldErrors("name"))`.

---

## medium/OrderSetDAO.java

- **Package:** `org.openmrs.api.db`
- **Type:** Public interface `OrderSetDAO`
- **Imports needed:** `java.util.List`, `org.openmrs.OrderSet`, `org.openmrs.OrderSetAttribute`, `org.openmrs.OrderSetAttributeType`, `org.openmrs.OrderSetMember`, `org.openmrs.api.OrderSetService`
- **Purpose:** Defines the data-access contract for `OrderSet`-related persistence, meant to be used only through `OrderSetService`, never directly.
- **Interface javadoc:** State it's OrderSet-related database functions and should never be used directly, only via `org.openmrs.api.OrderSetService`.
- **Method signatures (all throw `DAOException` unless noted, `DAOException` from `org.openmrs.api.db`):**
  - `OrderSet save(OrderSet orderSet) throws DAOException;`
  - `List<OrderSet> getOrderSets(boolean includeRetired) throws DAOException;`
  - `OrderSet getOrderSetById(Integer orderSetId) throws DAOException;`
  - `OrderSet getOrderSetByUniqueUuid(String orderSetUuid) throws DAOException;`
  - `OrderSetMember getOrderSetMemberByUuid(String uuid) throws DAOException;`
  - `List<OrderSetAttributeType> getAllOrderSetAttributeTypes();` (no throws clause)
  - `OrderSetAttributeType getOrderSetAttributeType(Integer id);`
  - `OrderSetAttributeType getOrderSetAttributeTypeByUuid(String uuid);`
  - `OrderSetAttributeType saveOrderSetAttributeType(OrderSetAttributeType orderSetAttributeType);`
  - `void deleteOrderSetAttributeType(OrderSetAttributeType orderSetAttributeType);`
  - `OrderSetAttribute getOrderSetAttributeByUuid(String uuid);`
  - `OrderSetAttributeType getOrderSetAttributeTypeByName(String name);`
  - Each method has a one-line javadoc `@see` tag pointing to the corresponding method on `OrderSetService`.

---

## medium/PersonVoidHandler.java

- **Package:** `org.openmrs.api.handler`
- **Type:** Public class `PersonVoidHandler`, implements `VoidHandler<Person>`
- **Annotation:** `@Handler(supports = Person.class)`
- **Imports needed:** `java.util.Date`, `org.openmrs.Person`, `org.openmrs.User`, `org.openmrs.annotation.Handler`, `org.openmrs.aop.RequiredDataAdvice`, `org.openmrs.api.UserService`, `org.openmrs.api.context.Context`
- **Purpose:** Sets the `personVoid*` fields on a `Person` when it is voided. Distinct from a generic `BaseVoidHandler` because `Person` uses `personVoided*`-prefixed fields rather than the standard `voided*` fields.
- **Method `handle(Person person, User voidingUser, Date voidedDate, String voidReason)`:** `@Override`, `void`.
  - If `!person.getPersonVoided()` (i.e. not already voided):
    - If `person.getPersonId() != null` (person is persisted), fetch `UserService us = Context.getUserService()`, loop over `us.getUsersByPerson(person, false)`, and call `us.retireUser(user, voidReason)` for each associated `User` account.
    - Set `person.setPersonVoided(true)`.
    - Set `person.setPersonVoidReason(voidReason)`.
    - If `person.getPersonVoidedBy() == null`, set it to `voidingUser`.
    - If `person.getPersonDateVoided() == null`, set it to `voidedDate`.
  - If the person is already voided, do nothing (idempotent no-op).

---

## medium/StartupErrorFilterTest.java

- **Package:** `org.openmrs.web.filter.startuperror`
- **Type:** JUnit 5 test class `StartupErrorFilterTest` (plain class, no special base class)
- **Imports needed:** `org.junit.jupiter.api.AfterEach`, `org.junit.jupiter.api.BeforeEach`, `org.junit.jupiter.api.Test`, `org.openmrs.web.Listener`, `org.springframework.mock.web.MockHttpServletRequest`, `org.springframework.test.util.ReflectionTestUtils`, static `org.hamcrest.CoreMatchers.is`, static `org.hamcrest.MatcherAssert.assertThat`, static `assertFalse`, static `assertTrue`
- **Purpose:** Tests `StartupErrorFilter`, a servlet filter that intercepts requests when OpenMRS failed to start correctly, redirecting to an error page instead of proceeding normally.
- **Field:** `private StartupErrorFilter filter;`
- **`@BeforeEach setUp()`:** instantiate `filter = new StartupErrorFilter();`
- **`@AfterEach reverterrorAtStartup()`:** use `ReflectionTestUtils.setField(Listener.class, "errorAtStartup", null)` to reset the static error state on the `Listener` class after each test (since it's a static field simulating app startup failure).
- **Method `getModel_shouldReturnAStartupErrorFilterModelContainingTheStartupError()`:** `@Test`, `void`.
  - Create an `Exception e`, set it via reflection onto `Listener.errorAtStartup`.
  - Call `filter.getUpdateFilterModel()` to get a `StartupErrorFilterModel model`.
  - Assert `model.errorAtStartup` is the same instance `e` (`assertThat(model.errorAtStartup, is(e))`).
- **Method `skipFilter_shouldReturnTrueIfNoErrorHasOccuredOnStartup()`:** `@Test`, `void`.
  - Assert `filter.skipFilter(new MockHttpServletRequest())` is true when there's no startup error, with message "should be true on start without error".
- **Method `skipFilter_shouldReturnFalseIfAnErrorHasOccuredOnStartup()`:** `@Test`, `void`.
  - Set a startup error via reflection, then assert `filter.skipFilter(...)` is false, with message "should be false on start with error".

---

## large/BaseAttribute.java

- **Package:** `org.openmrs.attribute`
- **Type:** Public abstract generic class `BaseAttribute<AT extends AttributeType, OwningType extends Customizable<?>>`, extends `BaseChangeableOpenmrsData`, implements `Attribute<AT, OwningType>` and `Comparable<Attribute>`
- **Annotations on class:** `@SuppressWarnings("rawtypes")`, `@MappedSuperclass` (JPA), `@Audited` (Hibernate Envers, for change auditing)
- **Imports needed:** `jakarta.persistence.Column`, `jakarta.persistence.JoinColumn`, `jakarta.persistence.ManyToOne`, `jakarta.persistence.MappedSuperclass`, `org.hibernate.envers.Audited`, `org.hibernate.search.mapper.pojo.mapping.definition.annotation.FullTextField`, `org.openmrs.BaseChangeableOpenmrsData`, `org.openmrs.customdatatype.CustomDatatypeUtil`, `org.openmrs.customdatatype.Customizable`, `org.openmrs.customdatatype.InvalidCustomValueException`, `org.openmrs.customdatatype.NotYetPersistedException`, `org.openmrs.util.OpenmrsUtil`
- **Purpose:** Abstract base implementation of `Attribute` intended so that concrete attribute subclasses (e.g. `PersonAttribute`, `VisitAttribute`) need almost no code of their own — this class handles owner/type storage, typed-value caching, dirty tracking, and comparison.
- **Fields:**
  - `@ManyToOne @JoinColumn(name = "owner_id", nullable = false) private OwningType owner;`
  - `@ManyToOne @JoinColumn(name = "attribute_type_id", nullable = false) private AT attributeType;`
  - `@FullTextField @Column(name = "value_reference", nullable = false, length = 65535) private String valueReference;` — the raw persisted value, with a comment noting it's "value pulled from the database".
  - `private transient Object value;` — a comment explains this temporarily holds the typed value, either lazily converted from `valueReference` on first `getValue()` call, or set directly via `setValue()` before the entity is persisted.
  - `private transient boolean dirty = false;`
- **Method `getOwner()`:** `@Override`, returns `owner`.
- **Method `setOwner(OwningType owner)`:** `@Override`, sets the field.
- **Method `setAttributeType(AT attributeType)`:** plain setter (not from the interface, has its own javadoc `@param`).
- **Method `getAttributeType()`:** `@Override`, returns `attributeType`.
- **Method `getDescriptor()`:** `@Override` (from `SingleCustomValue`), simply returns `getAttributeType()`.
- **Method `getValueReference()`:** `@Override`. If `valueReference == null`, throw `new NotYetPersistedException()`; otherwise return it.
- **Method `setValueReferenceInternal(String valueReference)`:** `@Override`, throws `InvalidCustomValueException`. Sets the field and resets `dirty = false`.
- **Method `getValue()`:** `@Override`, throws `InvalidCustomValueException`, returns `Object`. If `value == null`, lazily compute it via `CustomDatatypeUtil.getDatatype(getAttributeType()).fromReferenceString(getValueReference())` and cache it; return `value`.
- **Method `setValue(T typedValue)`:** generic `<T>`, `@Override`, throws `InvalidCustomValueException`. Sets `dirty = true` and `value = typedValue`.
- **Method `isDirty()`:** `@Override`, returns `dirty`.
- **Method `compareTo(Attribute other)`:** `@SuppressWarnings("squid:S1210")`, `@Override`. If `other == null` return `-1`. Otherwise compare in sequence, stopping at the first non-zero result: (1) `getVoided().compareTo(other.getVoided())`, (2) attribute-type IDs via `OpenmrsUtil.compareWithNullAsGreatest`, (3) value references via the same null-safe comparator, (4) entity IDs the same way. Return the final comparison result. Javadoc notes this comparator is inconsistent with `equals`.

---

## large/HibernateCohortDAO.java

- **Package:** `org.openmrs.api.db.hibernate`
- **Type:** Public class `HibernateCohortDAO`, implements `CohortDAO`
- **Annotation:** `@Repository("cohortDAO")`
- **Imports needed:** `java.util.ArrayList`, `java.util.Date`, `java.util.List`, `jakarta.persistence.criteria.CriteriaBuilder`, `jakarta.persistence.criteria.CriteriaQuery`, `jakarta.persistence.criteria.Join`, `jakarta.persistence.criteria.Predicate`, `jakarta.persistence.criteria.Root`, `org.hibernate.Session`, `org.hibernate.SessionFactory`, `org.openmrs.Cohort`, `org.openmrs.CohortMembership`, `org.openmrs.api.db.CohortDAO`, `org.openmrs.api.db.DAOException`, `org.springframework.beans.factory.annotation.Autowired`, `org.springframework.stereotype.Repository`
- **Purpose:** Hibernate JPA-Criteria-based implementation of the `CohortDAO` interface, backing `CohortService`.
- **Field:** `private static final String VOIDED = "voided";` (constant for the repeated `"voided"` property name), and `private final SessionFactory sessionFactory;`
- **Constructor:** `@Autowired public HibernateCohortDAO(SessionFactory sessionFactory)` — assigns the field.
- **Method `getCohort(Integer id)`:** `@Override`, throws `DAOException`. Returns `sessionFactory.getCurrentSession().get(Cohort.class, id)`.
- **Method `getCohortsContainingPatientId(Integer patientId, boolean includeVoided, Date asOfDate)`:** `@Override`, throws `DAOException`, returns `List<Cohort>`.
  - Build a `CriteriaQuery<Cohort>` rooted at `Cohort`, join to `"memberships"` (a `CohortMembership` collection) via `root.join("memberships")`.
  - Build a `List<Predicate>`. If `asOfDate != null`: add a predicate that `membershipJoin.get("startDate") <= asOfDate`, and add an OR predicate that `endDate` is null OR `endDate > asOfDate`.
  - Always add a predicate `membershipJoin.get("patientId") == patientId`.
  - If `!includeVoided`, add a predicate `root.get(VOIDED) == includeVoided` (i.e. `voided == false`).
  - Set `cq.distinct(true).where(predicates as array)`, execute and return the result list.
- **Method `getCohortByUuid(String uuid)`:** `@Override`. Delegates to a helper `HibernateUtil.getUniqueEntityByUUID(sessionFactory, Cohort.class, uuid)`.
- **Method `getCohortMembershipByUuid(String uuid)`:** `@Override`. Same pattern for `CohortMembership.class`.
- **Method `deleteCohort(Cohort cohort)`:** `@Override`, throws `DAOException`, returns `Cohort`. Calls `sessionFactory.getCurrentSession().remove(cohort)` and returns `null`.
- **Method `getCohorts(String nameFragment)`:** `@Override`, throws `DAOException`, returns `List<Cohort>`. Builds a criteria query where `lower(name) LIKE` a "contains" pattern built from `MatchMode.ANYWHERE.toLowerCasePattern(nameFragment)`, ordered ascending by `name`.
- **Method `getAllCohorts(boolean includeVoided)`:** `@Override`, throws `DAOException`, returns `List<Cohort>`. If `!includeVoided`, filter `voided == false`; always order ascending by `name`.
- **Method `getCohort(String name)`:** `@Override`, returns `Cohort`. Query where `name` equals the given name and `voided == false`; return `uniqueResult()`.
- **Method `saveCohort(Cohort cohort)`:** `@Override`, throws `DAOException`, returns `Cohort`. Delegates to `HibernateUtil.saveOrUpdate(sessionFactory.getCurrentSession(), cohort)`.
- **Method `getCohortMemberships(Integer patientId, Date activeOnDate, boolean includeVoided)`:** `@Override`, returns `List<CohortMembership>`.
  - Build predicates: always `patientId == patientId`.
  - If `activeOnDate != null`: `startDate <= activeOnDate` AND (`endDate` is null OR `endDate >= activeOnDate`).
  - If `!includeVoided`: `voided == false`.
  - Apply all predicates and return the result list.
- **Method `saveCohortMembership(CohortMembership cohortMembership)`:** `@Override`, returns `CohortMembership`. Delegates to `HibernateUtil.saveOrUpdate(...)`.

---

## large/VisitSearchCriteriaBuilder.java

- **Package:** `org.openmrs.parameter`
- **Type:** Public class `VisitSearchCriteriaBuilder` (classic fluent builder pattern)
- **Imports needed:** `java.util.Collection`, `java.util.Collections`, `java.util.Date`, `java.util.Map`, `org.openmrs.Concept`, `org.openmrs.Location`, `org.openmrs.Patient`, `org.openmrs.VisitAttributeType`, `org.openmrs.VisitType`
- **Purpose:** Builds `VisitSearchCriteria` instances via chained setter calls instead of a large constructor.
- **Fields (all private, no defaults except two flags):**
  - `Collection<VisitType> visitTypes;`
  - `Collection<Patient> patients;`
  - `Collection<Location> locations;`
  - `Collection<Concept> indications;`
  - `Date minStartDatetime;`
  - `Date maxStartDatetime;`
  - `Date minEndDatetime;`
  - `Date maxEndDatetime;`
  - `Map<VisitAttributeType, String> serializedAttributeValues;`
  - `boolean includeInactive = true;`
  - `boolean includeVoided = false;`
- **Constructor:** empty, no-arg `public VisitSearchCriteriaBuilder()`.
- **Fluent setter methods** (each sets the field and `return this;`, each returning `VisitSearchCriteriaBuilder`, each documented with a one-line javadoc `@param`/`@return`):
  - `visitTypes(Collection<VisitType> visitTypes)`
  - `patients(Collection<Patient> patients)`
  - `patient(Patient patient)` — convenience overload that sets `patients` to `Collections.singletonList(patient)`.
  - `locations(Collection<Location> locations)`
  - `indications(Collection<Concept> indications)`
  - `minStartDatetime(Date minStartDatetime)`
  - `maxStartDatetime(Date maxStartDatetime)`
  - `minEndDatetime(Date minEndDatetime)`
  - `maxEndDatetime(Date maxEndDatetime)`
  - `serializedAttributeValues(Map<VisitAttributeType, String> serializedAttributeValues)`
  - `includeInactive(boolean includeInactive)`
  - `includeVoided(boolean includeVoided)`
- **Method `build()`:** returns `new VisitSearchCriteria(...)`, passing all eleven fields positionally in this order: visitTypes, patients, locations, indications, minStartDatetime, maxStartDatetime, minEndDatetime, maxEndDatetime, serializedAttributeValues, includeInactive, includeVoided.
- **Class javadoc:** mentions `@since 2.6.8` and `@since 2.7.0` (backported to two release lines).

---

## large/WebUtilTest.java

- **Package:** `org.openmrs.web`
- **Type:** JUnit 5 test class `WebUtilTest` (plain class, no base class)
- **Imports needed:** `java.io.UnsupportedEncodingException`, `java.util.Collection`, `java.util.Locale`, `org.junit.jupiter.api.Test`, `org.openmrs.BaseOpenmrsObject`, static `assertEquals`, static `assertNull`
- **Purpose:** Tests three static utility methods on `WebUtil`: `getContextPath()`, `normalizeLocale(String)`, and `sanitizeLocales(String)`; also defines a small standalone test helper method.
- **`getContextPath` tests:**
  - `getContextPath_shouldReturnEmptyStringWhenWebAppNameIsNull()` — set `WebConstants.WEBAPP_NAME = null`, assert `WebUtil.getContextPath()` equals `""`.
  - `getContextPath_shouldReturnEmptyStringWhenWebAppNameIsEmptyString()` — set it to `""`, expect `""`.
  - `getContextPath_shouldReturnValueSpecifiedInWebAppName()` — set it to `"Value"`, expect `"/Value"` (prefixed with a slash).
- **`normalizeLocale` tests** (each calls the static method and checks the returned `Locale`):
  - `normalizeLocale_shouldIgnoreLeadingSpaces()` — `" it"` → `Locale.ITALIAN`.
  - `normalizeLocale_shouldAcceptLanguageOnlyLocales()` — `"fr"` → `Locale.FRENCH`.
  - `normalizeLocale_shouldNotAcceptInvalidLocales()` — `"ptrg"` and `"usaa"` both → `null`.
  - `normalizeLocale_shouldNotFailWithEmptyStrings()` — `""` → `null`.
  - `normalizeLocale_shouldNotFailWithWhitespaceOnly()` — `"      "` → `null`.
  - `normalizeLocale_shouldNotFailWithTab()` — throws `UnsupportedEncodingException`; build a string from byte `0x9` (tab) using `new String(bytes, "ASCII")`, expect `null`.
  - `normalizeLocale_shouldNotFailWithUnicode()` — `"Ši"` → `null`.
  - `normalizeLocale_shouldNotFailWithSingleChar()` — `"s"` → `null`.
  - `normalizeLocale_shouldNotFailWithUnderline()` — throws `UnsupportedEncodingException`; byte `0x5f` (underscore) → `null`.
- **`sanitizeLocales` tests:**
  - `sanitizeLocales_shouldSkipOverInvalidLocales()` — input `"és, qqqq, fr_RW, it, enñ"` (mix of valid and invalid locale tags, some with stray accented characters) should return `"fr_RW, it, en"` (only valid ones survive, cleaned of stray characters).
  - `sanitizeLocales_shouldNotFailWithEmptyString()` — `""` → `null` (note: uses `assertNull(null, WebUtil.sanitizeLocales(""))`, effectively asserting the result is null).
- **Static helper method `containsId(Collection<? extends BaseOpenmrsObject> list, Integer id)`:** `public static boolean`. Loops through the list; if any element's `getId()` equals the given `id`, return `true`; otherwise return `false` after the loop. Has its own javadoc describing params and return value.

---

## large/ConceptReferenceTermValidatorTest.java

- **Package:** `org.openmrs.validator`
- **Type:** JUnit 5 test class `ConceptReferenceTermValidatorTest`, extends `BaseContextSensitiveTest`
- **Imports needed:** `java.util.LinkedHashSet`, `java.util.Set`, `org.junit.jupiter.api.Disabled`, `org.junit.jupiter.api.Test`, `org.openmrs.ConceptMapType`, `org.openmrs.ConceptReferenceTerm`, `org.openmrs.ConceptReferenceTermMap`, `org.openmrs.api.ConceptService`, `org.openmrs.api.context.Context`, `org.openmrs.test.jupiter.BaseContextSensitiveTest`, `org.springframework.validation.BindException`, `org.springframework.validation.Errors`, static `assertFalse`, static `assertThrows`, static `assertTrue`
- **Purpose:** Comprehensive tests for `ConceptReferenceTermValidator.validate(Object, Errors)`, covering required-field checks, duplicate-code/name detection scoped by concept source, field-length limits, self-mapping prevention, and duplicate-mapping detection.
- Each test builds a `ConceptReferenceTerm`, sets some subset of `name`, `code`, `conceptSource` (fetched via `Context.getConceptService().getConceptSource(1)`), wraps it in `Errors errors = new BindException(term, "term")`, runs `new ConceptReferenceTermValidator().validate(term, errors)`, then asserts on `errors`. The specific test methods and their intent:
  - `validate_shouldFailIfTheCodeIsAWhiteSpaceCharacter()` — code set to `" "` → expect field error on `"code"`.
  - `validate_shouldFailIfTheCodeIsAnEmptyString()` — code `""` → field error on `"code"`.
  - `validate_shouldFailIfTheCodeIsNull()` — code never set (stays null) → field error on `"code"`.
  - `validate_shouldFailIfTheConceptReferenceTermCodeIsADuplicateInItsConceptSource()` — code `"WGT234"` (a code presumed to already exist for source 1 in test fixture data) → field error on `"code"`.
  - `validate_shouldFailIfTheConceptReferenceTermObjectIsNull()` — call `validate(null, errors)` and assert it throws `IllegalArgumentException` via `assertThrows`.
  - `validate_shouldFailIfTheConceptSourceIsNull()` — name and code set, conceptSource left null → field error on `"conceptSource"`.
  - `validate_shouldFailIfTheNameIsAWhiteSpaceCharacter()` — `@Disabled` with a comment "we might need these back when the constraint is put back"; name `" "` → would expect field error on `"name"`.
  - `validate_shouldFailIfTheNameIsAnEmptyString()` — `@Disabled`; name `""` → would expect field error on `"name"`.
  - `validate_shouldFailIfTheNameIsNull()` — `@Disabled`; name never set → would expect field error on `"name"`.
  - `validate_shouldPassIfAllTheRequiredFieldsAreSetAndValid()` — name, code, conceptSource all valid → `assertFalse(errors.hasErrors())`.
  - `validate_shouldPassIfTheDuplicateCodeIsForATermFromAnotherConceptSource()` — code `"2332523"` presumed duplicate globally but unique within source 1 → passes with no errors.
  - `validate_shouldPassIfTheDuplicateNameIsForATermFromAnotherConceptSource()` — name `"weight term2"` presumed duplicate globally but fine within source 1 → passes with no errors.
  - `validate_shouldFailIfAConceptReferenceTermMapHasNoConceptMapType()` — add a `ConceptReferenceTermMap` built with `(new ConceptReferenceTerm(1), null)` (null map type) via `term.addConceptReferenceTermMap(...)` → field error on `"conceptReferenceTermMaps[0].conceptMapType"`.
  - `validate_shouldFailIfTermBOfAConceptReferenceTermMapIsNotSet()` — build a `Set<ConceptReferenceTermMap>` (a `LinkedHashSet`) containing one map created as `(null, new ConceptMapType(1))` (null term B) and assign via `term.setConceptReferenceTermMaps(maps)` → field error on `"conceptReferenceTermMaps[0].termB"`.
  - `validate_shouldFailIfATermIsMappedToItself()` — fetch an existing term via `Context.getConceptService().getConceptReferenceTerm(1)`, take its first existing map, set that map's `termB` to point back to the term itself, re-set the maps, validate → field error on `"conceptReferenceTermMaps[0].termB"`.
  - `validate_shouldFailIfATermIsMappedMultipleTimesToTheSameTerm()` — build two separate `ConceptReferenceTermMap` objects both pointing to the same `(conceptReferenceTerm=1, conceptMapType=1)` pair, add both to the term → the second one (`conceptReferenceTermMaps[1].termB`) should get a field error; also prints `errors.getAllErrors()` to stderr for debugging.
  - `validate_shouldPassValidationIfFieldLengthsAreCorrect()` — set `name`, `code`, `conceptSource`, `version`, `description`, `retireReason` all to short valid strings → `assertFalse(errors.hasErrors())`.
  - `validate_shouldFailValidationIfFieldLengthsAreNotCorrect()` — set `name`, `code`, `version`, `description`, `retireReason` each to the same very long repeated string (the phrase "too long text " repeated ~20 times, several hundred characters) while `conceptSource` stays valid → assert field errors exist on all five of `"name"`, `"code"`, `"version"`, `"description"`, `"retireReason"`.
  - `validate_shouldPassIfTheConceptReferenceTermCodeIsDuplicateButRetired()` — code `"WGT234"` again, but first look up any existing term with that duplicate code via `getConceptReferenceTermByCode` and mark it `setRetired(true)` if found, so the duplicate check should be skipped for retired terms → assert `assertFalse(errors.hasFieldErrors("code"))`.
