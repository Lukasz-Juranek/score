..
   # *******************************************************************************
   # Copyright (c) 2025 Contributors to the Eclipse Foundation
   #
   # See the NOTICE file(s) distributed with this work for additional
   # information regarding copyright ownership.
   #
   # This program and the accompanying materials are made available under the
   # terms of the Apache License Version 2.0 which is available at
   # https://www.apache.org/licenses/LICENSE-2.0
   #
   # SPDX-License-Identifier: Apache-2.0
   # *******************************************************************************

GoogleTest (GTest/gMock) Verification Report
============================================

Introduction
------------

Scope and purpose
~~~~~~~~~~~~~~~~~

GoogleTest (GTest) is the C++ testing framework used for unit and component tests in
S-CORE C++ modules. gMock, distributed in the same package and the same Bazel module, is
used for mock objects, call expectations and actions.

S-CORE relies on GoogleTest for two purposes:

* deciding whether a test detected an error (assertions, mock expectations, death and
  exception tests), and
* reporting the result per test case, including the ``RecordProperty()`` values
  (``FullyVerifies``, ``PartiallyVerifies``, ``TestType``, ``DerivationTechnique``, ...)
  that docs-as-code turns into requirement-verification evidence
  (see :need:`gd_req__verification_link_tests_cpp`).

The evaluation covers the use of GoogleTest for software unit and integration
verification of safety-related elements up to **ASIL B** (ISO 26262-6 clauses 9 and 10),
in the intended environment below. Use for a higher ASIL or in another environment is
outside this evaluation.

Inputs and outputs
~~~~~~~~~~~~~~~~~~

| Inputs: Software sources (C++), GoogleTest/gMock-based test files (C++), command line
  and environment set by Bazel (``XML_OUTPUT_FILE``, sharding, filters).
| Outputs: Test binary, process exit code, test report (console, XML ``test.xml`` read by
  Bazel and docs-as-code).

.. figure:: ../_assets/gtest.drawio.svg
  :width: 100%
  :align: center
  :alt: GTest overview

  GTest overview

Available information
~~~~~~~~~~~~~~~~~~~~~

- Version: 1.17.0.bcr.2 (Bazel Central Registry module ``googletest``)
- Official repository: https://github.com/google/googletest
- Official documentation: https://google.github.io/googletest
- API Reference: https://google.github.io/googletest/reference/testing.html
- Known issues: https://github.com/google/googletest/issues

Installation and integration
----------------------------

Installation
~~~~~~~~~~~~

Add the evaluated version to the module's ``MODULE.bazel``:

.. code-block:: Python

  bazel_dep(name = "googletest", version = "1.17.0.bcr.2")

All S-CORE modules shall use this version for safety-related unit verification
(see constraint of use 5 below).

Integration
~~~~~~~~~~~

Integrated in Bazel through ``cc_test`` targets depending on ``@googletest//:gtest`` or
``@googletest//:gtest_main``. Bazel sets ``XML_OUTPUT_FILE``; GoogleTest writes the XML
report there, and docs-as-code reads it from ``bazel-testlogs``.

Intended environment
~~~~~~~~~~~~~~~~~~~~

The result depends on the whole tool configuration. The evaluation and its measures cover
the following configuration; every other configuration is outside this evaluation until
the measures run in it.

.. list-table:: Intended environment
   :header-rows: 1
   :widths: 3 5 5

   * - Element
     - Evaluated configuration
     - Measures executed
   * - GoogleTest/gMock
     - 1.17.0.bcr.2 (BCR), ``gtest_main``/``gmock_main``
     - M1, M2, M3
   * - Build system
     - Bazel 8.6.0, ``--lockfile_mode=error``
     - M1, M2
   * - Compiler and target (unit tests)
     - GCC 12.2.0, Bazel config ``x86_64-linux``, ``fastbuild``, warnings as errors
     - M1, M2 (CI of the reference module)
   * - Compiler and target (QNX)
     - QCC, SDP 8.0, ``arm64-qnx`` / ``x86_64-qnx``
     - **not covered** (open item)
   * - Fault injection
     - Clang 19.1.1 at ``-O0``, Mull 0.34.2, Ubuntu 24.04
     - M3 only
   * - Report consumer
     - docs-as-code 8.4.0 (``parse_testcase_result``)
     - M1 (parity test of the checker)

Intended usage concept
----------------------

The safety evaluation below assumes the following measures. They are part of the intended
usage of GoogleTest in S-CORE; a module that uses GoogleTest for safety-related unit
verification without them is outside this evaluation.

A reference implementation of measures 1 to 3 is provided for the logging module in
``quality/gtest_tool_confidence``.

#. **Canary suite (M1).** In every CI configuration that runs unit tests, a canary suite
   is built with the same toolchain, flags and GoogleTest version as the tests. For every
   GoogleTest/gMock feature in the feature inventory below it contains test cases that
   must fail and test cases that must pass, skip or be disabled; further binaries check
   the exit code with a single failure, failures outside a test case and a premature
   exit. An independent checker (not GoogleTest) verifies the exit code, the
   premature-exit marker, the presence of every listed test case in the XML report and
   vice versa, the reported result of every test case and the ``RecordProperty()``
   values, including that no test case reports a property it did not record. Any
   deviation fails CI. The checker has its own unit tests, which show that it fails for
   every simulated malfunction, and a parity test, which runs the docs-as-code parser on
   the canary reports.

   M1 detects systematic GoogleTest defects in the tool configuration. It does not use
   the inputs of the real tests, so it does not detect a defect that only shows with
   particular values (see limitations). M1 is the **validation suite** for the
   qualification of GoogleTest; it is not by itself sufficient as detection measure.
#. **Test-run completeness check (M2).** After every unit-test run, every test target
   in scope must have a report; for each GoogleTest target (shards merged), the report
   is readable, contains test cases, its declared test count matches, no test case has
   an ``<error>`` or is reported as failed after a green run, no failure outside a test
   case is reported, the tests listed by the binary (``--gtest_list_tests``) and the
   reported tests are identical, a target that needed more than one attempt is reported,
   and skipped or disabled test cases that declare requirement links are errors
   (``--fail-on-skipped-verification``).
#. **GoogleTest fault injection (M3).** Centrally, once per GoogleTest version and per
   toolchain change (not per module), faults are injected into the compiled
   GoogleTest/gMock library sources (Mull operators: arithmetic, comparison, boundary,
   logical, removed void call, replaced scalar call) and the canary checker is run
   against every faulty GoogleTest, one fault at a time. This measures how well M1
   detects real GoogleTest defects.

   Every undetected fault is classified as *equivalent* (the mutated code cannot change
   a verdict, exit code, test list or XML report in the intended usage; the justification
   names the code path and is confirmed by a second person) or as a *gap* in the canary
   suite, which is closed by a new canary and a re-run. Acceptance criterion: no
   unclassified fault and no open gap. The detection rate is reported, not gated.

   Status: the campaign with GoogleTest 1.17.0.bcr.2 found 1,710 mutants. Their
   execution and classification are **pending**; no claim of this report relies on M3.
#. **CI status check (M4).** A failing, crashing or timed-out test target fails the CI job;
   the job result is part of the release checklist.
#. **Single evaluated version (M5).** All modules use the GoogleTest version named in this
   report. A version change requires an update of this report and a canary run. M5 is
   currently a constraint of use without automated enforcement (open item).

Feature inventory
~~~~~~~~~~~~~~~~~

The canary suite covers the following features (82 listed canary test cases in six
binaries in the reference implementation). Safety-related tests shall use only features of
this inventory; a module that needs another feature adds a canary first.

.. list-table:: GoogleTest/gMock feature inventory
   :header-rows: 1
   :widths: 5 6 4

   * - Feature
     - Canaries (must fail / must pass)
     - Tool requirement
   * - ``EXPECT_EQ/NE/LT/LE/GT/GE/TRUE/FALSE``, ``EXPECT_FLOAT_EQ``, ``EXPECT_DOUBLE_EQ``,
       ``EXPECT_NEAR``, ``EXPECT_STREQ``, ``EXPECT_PRED1/2``, ``ADD_FAILURE``, user-defined
       ``operator==``; boundary inputs (64-bit extremes, long strings, embedded NUL, long
       containers, 4/5 ULP, NaN); failure in another thread
     - both
     - :need:`tool_req__gtest_nonfatal_failure_reported`
   * - Matchers with ``EXPECT_THAT``/``ASSERT_THAT``: ``Eq``, ``ElementsAre``,
       ``UnorderedElementsAre``, ``Contains``, ``HasSubstr``, ``AllOf``, ``Not``, ``Gt``,
       ``Lt``, ``IsEmpty``, ``Pointee``, ``Field``
     - both
     - :need:`tool_req__gtest_nonfatal_failure_reported`
   * - ``ASSERT_EQ``, ``ASSERT_TRUE``, ``ASSERT_THAT``, ``FAIL``, fatal failure in a helper
     - must fail
     - :need:`tool_req__gtest_fatal_failure_reported`
   * - ``EXPECT_THROW``, ``EXPECT_ANY_THROW``, ``EXPECT_NO_THROW``, uncaught exception
     - both
     - :need:`tool_req__gtest_exception_failure_reported`
   * - Fixtures: ``SetUp``, ``TearDown``, ``SetUpTestSuite``, ``TearDownTestSuite``, global
       ``Environment``
     - must fail
     - :need:`tool_req__gtest_fixture_failure_reported`
   * - ``TEST_P`` with ``INSTANTIATE_TEST_SUITE_P``, ``TYPED_TEST``
     - per instance
     - :need:`tool_req__gtest_generated_tests_reported`
   * - ``EXPECT_DEATH``, ``ASSERT_DEATH``, ``EXPECT_EXIT``
     - both
     - :need:`tool_req__gtest_death_test_reported`
   * - ``EXPECT_CALL`` with ``Times``, argument matchers, ``StrictMock``, ``NiceMock``,
       default mock, ``InSequence``
     - both
     - :need:`tool_req__gtest_mock_expectation_verified`
   * - ``Return``, ``WillOnce``, ``WillRepeatedly``, ``Invoke``, ``SetArgPointee``,
       ``ON_CALL(...).WillByDefault``, default return value
     - must pass
     - :need:`tool_req__gtest_mock_actions_performed`
   * - ``GTEST_SKIP`` (test body and ``SetUp``), ``DISABLED_``
     - skipped / disabled
     - :need:`tool_req__gtest_skip_disabled_reported`
   * - ``RecordProperty`` (``FullyVerifies``, ``PartiallyVerifies``, ``Verifies``,
       ``Description``, ``TestType``, ``DerivationTechnique``, ``ASIL``)
     - must pass
     - :need:`tool_req__gtest_record_property_reported`
   * - Exit code (many failures, a single failure, all passed), premature ``exit(0)``
     - both
     - :need:`tool_req__gtest_exit_code`, :need:`tool_req__gtest_report_complete`

Limitations of the measures
~~~~~~~~~~~~~~~~~~~~~~~~~~~

#. **Canaries are not the real tests.** M1 uses chosen inputs, including boundary values.
   A GoogleTest defect that only shows with the values, types or call sequences of a real
   test is not detected by M1. Detection measures that act on every real test run are
   M2 and M4; they detect lost, incomplete or inconsistent results, not wrong verdicts.
#. **M3 scope.** Only the compiled GoogleTest/gMock library sources (``googletest/src``,
   ``googlemock/src``) are mutated. Header-only code is instantiated in the canary sources
   and is not mutated: the assertion helpers (``CmpHelperEQ`` and related templates), most
   matchers, the typed expectation and action code in ``gmock-spec-builders.h`` and
   ``gmock-actions.h``, and the parameterized and typed test machinery. M3 therefore says
   little about gMock expectations and actions and nothing about these headers.
#. **M3 toolchain.** M3 builds GoogleTest with Clang 19 at ``-O0``. The production builds
   use GCC 12 and QCC with other optimisation levels; compiler-specific code generation
   of GoogleTest is not fault-injected.
#. **QNX not covered.** M1 and M2 run only in the ``x86_64-linux`` configuration. Death
   tests, process creation and exit handling differ on QNX; until the canary suite runs
   there, GoogleTest results on QNX are outside this evaluation.
#. **Common cause in M2.** ``--gtest_list_tests`` and the XML report are produced by the
   same binary from the same test registry. A test that is never registered (for example
   a lost value-parameterized instance) is missing from both; only M1 detects it.
#. **Not enforced across modules.** M1 to M3 are implemented in one reference module.
   Nothing yet enforces that every module runs M1 and M2 or uses the evaluated version.

Report record
-------------

.. doc_tool:: GoogleTest
   :id: doc_tool__gtest
   :status: evaluated
   :version: 3
   :tool_version: 1.17.0.bcr.2
   :security_affected: YES
   :realizes: wp__tool_verification_report[version==1]
   :tags: tool_management, tools_testing_frameworks
   :post_template: tool_qualification_report

   Evaluates GoogleTest and gMock for executing C++ unit tests and reporting their
   results, including the requirement-verification properties used by docs-as-code.
   The evaluation results in SCORE TCL LOW: GoogleTest is to be qualified by
   validation, with the canary suite as validation suite (see "Tool qualification").

Safety evaluation
-----------------

Each use case below records how S-CORE relies on GoogleTest. The potential malfunctions
are evaluated against the intended usage concept above. ``detection_sufficient: YES`` is
claimed only where a measure acts on every real test run and detects the malfunction
independently of the canaries; where only the canaries (M1) detect it, detection is not
sufficient and the violated tool requirements are qualified by validation.

.. tool_usecase:: Decide pass/fail of assertions
   :id: tool_usecase__gtest__assertions
   :belongs_to: doc_tool__gtest
   :version: 1

   A unit test checks the behaviour of the code under test with GoogleTest assertions
   (``EXPECT_*``/``ASSERT_*``, matchers, ``FAIL``/``ADD_FAILURE``, exception assertions)
   in plain, fixture, value-parameterized and typed tests. A violated assertion must
   result in a failed test case.

   .. potential_tool_malfunction:: Violated assertion is not reported as failure
      :id: potential_tool_malfunction__gtest__false_pass
      :safety_affected: YES
      :detection_sufficient: NO
      :safety_measures: M1 canary suite per CI configuration (validation suite)
      :violates:
         tool_req__gtest_nonfatal_failure_reported,
         tool_req__gtest_fatal_failure_reported,
         tool_req__gtest_exception_failure_reported,
         tool_req__gtest_fixture_failure_reported
      :version: 2

      GoogleTest fails to detect an existing error: an assertion that is violated, an
      uncaught exception or a failure in a fixture does not mark the test case as failed,
      or a fatal assertion does not stop the test.

      M1 executes must-fail canaries for every assertion kind of the feature inventory in
      the tool configuration of the tests and detects a systematic defect. A defect that
      depends on the values of a real test is not detected (limitation 1), and the
      sensitivity of M1 to real GoogleTest defects is not yet measured (M3 pending).
      Detection is therefore not sufficient.

      A failure in ``TearDownTestSuite()`` leaves the suite's test cases reported as
      passed; the failure appears only as a nameless report entry and in the exit code.
      M4 and M2 detect it.

   .. potential_tool_malfunction:: Generated test instances are lost or merged
      :id: potential_tool_malfunction__gtest__gen_tests
      :safety_affected: YES
      :detection_sufficient: NO
      :safety_measures: M1 canary suite (per-instance results); M2 completeness check (reported instances only)
      :violates: tool_req__gtest_generated_tests_reported
      :version: 2

      Value-parameterized or typed test instances are not generated, not executed, or
      their results are attributed to the wrong instance, so a failing instance is hidden.

      M1 contains parameterized and typed canaries in which only one instance fails and
      checks every instance's result. M2 detects an instance that is listed but not
      reported; an instance that is never generated is missing from both the list and the
      report (limitation 5).

   .. potential_tool_malfunction:: Non-existing error is reported
      :id: potential_tool_malfunction__gtest__false_fail
      :safety_affected: NO
      :violates: tool_req__gtest_no_false_failure
      :version: 1

      GoogleTest reports a failure although the code under test is correct. The failure is
      visible and leads to analysis; it cannot hide an error. M1 must-pass canaries detect
      it as well.

.. tool_usecase:: Verify mock expectations and perform mock actions
   :id: tool_usecase__gtest__mocks
   :belongs_to: doc_tool__gtest
   :version: 2

   A unit test uses gMock (``EXPECT_CALL``, ``StrictMock``, ``NiceMock``, ``InSequence``,
   cardinalities, argument matchers) to check the interaction of the code under test with
   its dependencies, and gMock actions (``Return``, ``WillOnce``, ``WillRepeatedly``,
   ``Invoke``, ``SetArgPointee``, ``ON_CALL``) to control the values the dependencies
   return to the code under test. A violated expectation must result in a failed test
   case; an action must deliver the specified value or side effect.

   .. potential_tool_malfunction:: Violated mock expectation is not reported
      :id: potential_tool_malfunction__gtest__mock_pass
      :safety_affected: YES
      :detection_sufficient: NO
      :safety_measures: M1 canary suite per CI configuration (validation suite)
      :violates: tool_req__gtest_mock_expectation_verified
      :version: 2

      An unmet expectation, a call with unexpected arguments, too many calls, an
      uninteresting call on a ``StrictMock`` or a sequence violation is not reported.

      M1 contains a must-fail canary per kind of violation. Most of the expectation logic
      is header-only template code that M3 does not mutate (limitation 2).

   .. potential_tool_malfunction:: Mock action delivers a wrong value or side effect
      :id: potential_tool_malfunction__gtest__mock_act
      :safety_affected: YES
      :detection_sufficient: NO
      :safety_measures: M1 canary suite (action canaries) per CI configuration (validation suite)
      :violates: tool_req__gtest_mock_actions_performed
      :version: 1

      A mocked function returns another value than specified, an action is not performed
      or performed for the wrong call, or ``SetArgPointee`` does not write the output
      argument. The code under test then runs a different path than the test intends; a
      test written for an error path can pass on the nominal path, so the error path is
      reported as verified without having been executed.

      M1 checks the values delivered by each action in the feature inventory. The action
      code is header-only and not covered by M3 (limitation 2).

.. tool_usecase:: Death and exit tests
   :id: tool_usecase__gtest__death_tests
   :belongs_to: doc_tool__gtest
   :version: 1

   A unit test checks with ``EXPECT_DEATH``/``ASSERT_DEATH``/``EXPECT_EXIT`` that the code
   under test terminates (for example on a violated precondition) with the expected
   message or exit code.

   .. potential_tool_malfunction:: Death test passes without the expected termination
      :id: potential_tool_malfunction__gtest__death_pass
      :safety_affected: YES
      :detection_sufficient: NO
      :safety_measures: M1 canary suite in the x86_64-linux configuration (validation suite)
      :violates: tool_req__gtest_death_test_reported
      :version: 2

      A death test passes although the statement does not terminate, terminates with a
      different message, or exits with a different code. Death tests depend on the
      operating system (process creation); the canary suite runs only on Linux so far, so
      death tests on QNX are outside this evaluation (limitation 4).

.. tool_usecase:: Report test results to Bazel, CI and docs-as-code
   :id: tool_usecase__gtest__reporting
   :belongs_to: doc_tool__gtest
   :version: 1

   The process exit code tells Bazel and CI whether the test target passed. The XML report
   tells docs-as-code, per test case, whether it passed, failed, was skipped or disabled,
   and which requirements it verifies (``RecordProperty()``).

   .. potential_tool_malfunction:: Exit code indicates success despite failed tests
      :id: potential_tool_malfunction__gtest__exit_code
      :safety_affected: YES
      :detection_sufficient: YES
      :safety_measures: M2 completeness check reports every failed test case and every failure outside a test case after a green run; M1 checks the exit code with many and with a single failure
      :violates: tool_req__gtest_exit_code
      :version: 2

      The test binary exits with 0 although test cases failed, so Bazel and CI report the
      target as passed.

      M2 runs after every green test run and reads the per-test results from the XML
      report, which GoogleTest writes independently of the exit code; a failed test case
      or a failed suite-level entry is an error. The case in which the result itself is
      wrong is the malfunction "Violated assertion is not reported as failure".

   .. potential_tool_malfunction:: Test result is missing from the report
      :id: potential_tool_malfunction__gtest__lost_res
      :safety_affected: YES
      :detection_sufficient: NO
      :safety_measures: M2 completeness check (expected targets, list vs. report, counts); Bazel premature-exit marker checked by M1
      :violates: tool_req__gtest_report_complete
      :version: 2

      A test case is not executed, or its result is not written to the report (for
      example after a crash, with sharding or with filters), so a requirement appears
      verified by an incomplete run.

      M2 detects a target without report, a listed test without result, a reported test
      that is not listed, a filtered run and a lost shard. A test calling ``exit(0)`` ends
      the binary with exit code 0; GoogleTest leaves the file named by
      ``TEST_PREMATURE_EXIT_FILE`` behind and Bazel fails the target, and the report is
      missing or incomplete; M1 verifies both. A test that is never registered is missing
      from the list and the report alike (limitation 5), so detection is not sufficient.

   .. potential_tool_malfunction:: Skipped or disabled test is reported as passed
      :id: potential_tool_malfunction__gtest__skip_pass
      :safety_affected: YES
      :detection_sufficient: NO
      :safety_measures: M1 canary suite (skipped and disabled canaries); M2 completeness check fails on skipped/disabled tests with requirement links
      :violates: tool_req__gtest_skip_disabled_reported
      :version: 2

      docs-as-code treats a test case without ``<failure>``, ``<error>`` or ``<skipped>``
      element as passed. If GoogleTest omits the ``<skipped>`` element or the
      ``status="notrun"`` attribute, a test that verified nothing counts as verification
      evidence. M2 can only act on a test that is reported as skipped; the omission
      itself is detected only by M1.

   .. potential_tool_malfunction:: Requirement properties are wrong in the report
      :id: potential_tool_malfunction__gtest__props
      :safety_affected: YES
      :detection_sufficient: NO
      :safety_measures: M1 canary suite checks RecordProperty values and their test case; docs-as-code reports unresolved requirement references
      :violates: tool_req__gtest_record_property_reported
      :version: 2

      A ``RecordProperty()`` value (``FullyVerifies``, ``PartiallyVerifies``, ...) is lost,
      truncated or attached to the wrong test case, so the traceability shows a
      requirement as verified by the wrong test or not at all. docs-as-code detects a
      reference to a non-existing requirement, not a valid reference at the wrong test
      case.

   .. potential_tool_malfunction:: Report cannot be written or input cannot be loaded
      :id: potential_tool_malfunction__gtest__io
      :safety_affected: YES
      :detection_sufficient: YES
      :safety_measures: M4 CI status check; M2 completeness check (target without report, unreadable or non-GoogleTest report)
      :violates: tool_req__gtest_report_complete
      :version: 2

      GoogleTest fails to load a provided file or to write the report. The test target
      fails, or the report is missing, unreadable or replaced by Bazel's generated report;
      M4 and M2 make each case visible.

Tool requirements
-----------------

The capabilities of GoogleTest that S-CORE relies on. The canary suite (M1) contains at
least one canary per requirement.

.. tool_req:: Non-fatal assertion failures are reported
   :id: tool_req__gtest_nonfatal_failure_reported
   :version: 2
   :satisfies: gd_req__verification_reporting[version==1]

   A violated ``EXPECT_*`` assertion (including ``EXPECT_THAT`` with matchers,
   ``EXPECT_PRED*``, floating point and C string comparisons and user-defined
   ``operator==``) and ``ADD_FAILURE()`` shall mark the test case as failed, also when
   raised in another thread of the test.

.. tool_req:: Fatal assertion failures are reported and stop the test
   :id: tool_req__gtest_fatal_failure_reported
   :version: 1
   :satisfies: gd_req__verification_reporting[version==1]

   A violated ``ASSERT_*`` assertion and ``FAIL()`` shall mark the test case as failed and
   return from the current function; a fatal failure in a helper function shall fail the
   calling test case.

.. tool_req:: Exception-related failures are reported
   :id: tool_req__gtest_exception_failure_reported
   :version: 1
   :satisfies: gd_req__verification_reporting[version==1]

   An exception escaping the test body, ``EXPECT_THROW`` without or with a wrong exception,
   ``EXPECT_ANY_THROW`` without exception and ``EXPECT_NO_THROW`` with exception shall
   mark the test case as failed.

.. tool_req:: Fixture failures are reported
   :id: tool_req__gtest_fixture_failure_reported
   :version: 2
   :satisfies: gd_req__verification_reporting[version==1]

   A failure in a test fixture's ``SetUp()`` or ``TearDown()`` shall mark the test case as
   failed. A failure in ``SetUpTestSuite()``, ``TearDownTestSuite()`` or a global
   ``Environment`` shall make the binary exit with a non-zero code and appear as a failed
   entry in the XML report; test cases that were not executed because of it shall not be
   reported as passed.

.. tool_req:: Generated test instances are reported individually
   :id: tool_req__gtest_generated_tests_reported
   :version: 1
   :satisfies: gd_req__verification_reporting[version==1]

   Every instance of a value-parameterized (``TEST_P``) and typed (``TYPED_TEST``) test
   shall be executed and reported as a separate test case with its own result.

.. tool_req:: Death test expectations are verified
   :id: tool_req__gtest_death_test_reported
   :version: 1
   :satisfies: gd_req__verification_reporting[version==1]

   ``EXPECT_DEATH``/``ASSERT_DEATH`` shall fail if the statement does not terminate or its
   output does not match; ``EXPECT_EXIT`` shall fail if the exit code does not match.

.. tool_req:: Mock expectations are verified
   :id: tool_req__gtest_mock_expectation_verified
   :version: 1
   :satisfies: gd_req__verification_reporting[version==1]

   gMock shall mark the test case as failed on an unmet expectation, a call exceeding the
   cardinality, a call with arguments matching no expectation, an uninteresting call on a
   ``StrictMock`` and a violated ``InSequence`` order.

.. tool_req:: Mock actions are performed as specified
   :id: tool_req__gtest_mock_actions_performed
   :version: 1
   :satisfies: gd_req__verification_reporting[version==1]

   A mocked function shall return the value specified with ``Return`` for the call that
   ``WillOnce``/``WillRepeatedly`` assigns it to, shall call the function given to
   ``Invoke`` with the call's arguments, shall write the value given to
   ``SetArgPointee``, and shall use the ``ON_CALL`` default action or, without one, the
   default value of the return type.

.. tool_req:: Correct tests are not reported as failed
   :id: tool_req__gtest_no_false_failure
   :version: 1
   :satisfies: gd_req__verification_reporting[version==1]

   Satisfied assertions, death tests, mock expectations and uninteresting calls on a
   ``NiceMock`` shall not mark the test case as failed.

.. tool_req:: Skipped and disabled tests are reported as such
   :id: tool_req__gtest_skip_disabled_reported
   :version: 1
   :satisfies: gd_req__verification_reporting[version==1]

   A test case calling ``GTEST_SKIP()`` shall be reported with a ``<skipped>`` element and
   a ``DISABLED_`` test case shall not be executed and shall be reported with
   ``status="notrun"``.

.. tool_req:: Recorded properties reach the report
   :id: tool_req__gtest_record_property_reported
   :version: 2
   :satisfies:
      gd_req__verification_link_tests_cpp[version==1],
      gd_req__verification_reporting[version==1]

   Values recorded with ``RecordProperty()`` shall appear unchanged in the XML report at the
   test case that recorded them and at no other test case, and values recorded after a
   fatal failure shall not appear.

.. tool_req:: Exit code reflects the test result
   :id: tool_req__gtest_exit_code
   :version: 1
   :satisfies: gd_req__verification_reporting[version==1]

   A test binary shall exit with a non-zero code if at least one test case failed, and with
   zero if all executed test cases passed or were skipped.

.. tool_req:: Report is complete
   :id: tool_req__gtest_report_complete
   :version: 1
   :satisfies: gd_req__verification_reporting[version==1]

   The XML report shall contain exactly one result for every test case the binary lists
   with ``--gtest_list_tests``, and its declared test count shall match the reported test
   cases.

Constraints of use
------------------

#. Run the canary suite (M1) in every CI configuration that runs unit tests whose results
   are used as verification evidence. Configurations without M1 (currently all QNX
   configurations) are outside this evaluation.
#. Run the test-run completeness check (M2) after the unit tests with
   ``--expected-targets`` and ``--fail-on-skipped-verification``; treat its errors as test
   failures.
#. Before a GoogleTest version or toolchain change is used for verification evidence,
   repeat the GoogleTest fault-injection campaign (M3) centrally and archive its result.
#. Do not use ``GTEST_SKIP()`` or ``DISABLED_`` on test cases that declare
   ``FullyVerifies``/``PartiallyVerifies`` for a safety-related requirement.
#. Use the GoogleTest version named in this report (M5).
#. Do not set ``GTEST_FILTER``, ``--gtest_filter`` or ``--gtest_also_run_disabled_tests``
   in CI runs whose reports are used as verification evidence, do not unset
   ``TEST_PREMATURE_EXIT_FILE`` (set by Bazel), and do not use ``flaky_test_attempts`` or
   ``flaky = True`` for such tests (a failed attempt is not visible in the final report).
#. To detect unexpected calls, use ``StrictMock`` or ``EXPECT_CALL(...).Times(0)``. A
   default ("naggy") mock only prints a warning for an uninteresting call and does not
   fail the test; the canary suite pins this behaviour.
#. Use only GoogleTest/gMock features of the feature inventory in safety-related tests;
   add a canary before using another feature.
#. Do not rely on failures in ``SetUpTestSuite()``, ``TearDownTestSuite()`` or a global
   ``Environment`` to fail a requirement's test case: GoogleTest reports them outside the
   test cases.

Security evaluation
-------------------

GoogleTest is linked only into test binaries (``testonly``) and is not part of any
delivered software. The security-relevant asset is the integrity of the verification
evidence produced with it.

.. list-table:: Security evaluation
   :header-rows: 1

   * - Threat identification
     - Use case description
     - Threats
     - Impact on security?
     - Impact security measures available?
     - Impact security detection sufficient?
   * - 1
     - Obtain GoogleTest from the Bazel Central Registry
     - Tampered or substituted GoogleTest sources that hide test failures
     - YES (integrity of verification evidence)
     - YES: version pinned in ``MODULE.bazel``, source integrity hash in
       ``MODULE.bazel.lock`` enforced with ``--lockfile_mode=error``; M1 detects a
       changed verdict behaviour
     - YES
   * - 2
     - Run GoogleTest test binaries in CI
     - Code in a test binary attacks the CI runner
     - NO (test code is reviewed like production code; runners are isolated GitHub
       runners)
     - YES: code review, repository permissions
     - YES
   * - 3
     - Fault-injection campaign (M3)
     - Tampered Mull package or CI actions falsify the M3 evidence
     - YES (integrity of M3 evidence only)
     - YES: Mull package checked against its SHA-256, CI actions pinned by commit SHA
     - YES

Result
~~~~~~

GoogleTest is safety-affected. With the measures implemented today, detection is
sufficient only for a wrong exit code and for a report that cannot be written; for the
other safety-relevant malfunctions only the canary suite detects a defect, and only a
systematic one. The derived **SCORE TCL is LOW**: GoogleTest requires qualification.

Tool qualification
------------------

Based on method: validation of the software tool (ISO 26262-8, 11.4.9), in the intended
environment above.

* Validation suite: the canary suite (M1) with its independent checker, executed in each
  CI configuration of the intended environment. Each tool requirement violated by a
  malfunction with insufficient detection is validated by the canaries named in the
  feature inventory.
* Evidence: the checker's per-canary evidence table (``gtest_canary_evidence.md``) and
  the CI result per configuration. For this report to reach ``qualified``, the canaries
  shall produce ``testcase`` needs that link to the tool requirements with
  ``fully_verifies``.
* Sensitivity of the validation suite: M3, with the acceptance criterion above.

Open items
~~~~~~~~~~

#. Link the canaries to the tool requirements (``FullyVerifies``) and make their results
   available to this documentation; then set the status to ``qualified``.
#. Execute and archive the M3 campaign with every undetected fault classified.
#. Run M1 and M2 in the QNX configurations (QEMU test runner).
#. Move M1 to M3 into a central package (for example ``score_tooling``) and enforce in CI
   that every module using GoogleTest runs M1 and M2 and uses the evaluated version (M5).
#. Classify the surviving mutants of the optional production-code mutation run of the
   reference module.
#. docs-as-code asserts that every ``<testcase>`` has a ``name`` or ``classname``;
   GoogleTest's entry for a failure outside a test case has neither. Align the reader.
#. Confirmation review of this report by an independent assessor before ``released``.
