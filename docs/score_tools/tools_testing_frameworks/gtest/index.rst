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
used for mock objects and call expectations.

S-CORE relies on GoogleTest for two purposes:

* deciding whether a test detected an error (assertions, mock expectations, death and
  exception tests), and
* reporting the result per test case, including the ``RecordProperty()`` values
  (``FullyVerifies``, ``PartiallyVerifies``, ``TestType``, ``DerivationTechnique``, ...)
  that docs-as-code turns into requirement-verification evidence
  (see :need:`gd_req__verification_link_tests_cpp`).

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

Environment
~~~~~~~~~~~

Requires a C++ compiler and the Bazel build environment. The result depends on the whole
tool configuration: GoogleTest version, compiler and version (GCC, Clang, QCC), compiler
flags (optimisation, exceptions, sanitizers), target OS (Linux, QNX), Bazel version and the
docs-as-code XML reader.

Intended usage concept
----------------------

The safety evaluation below assumes the following measures. They are part of the intended
usage of GoogleTest in S-CORE; a module that uses GoogleTest for safety-related unit
verification without them is outside this evaluation.

A reference implementation of measures 1 to 3 is provided for the logging module in
``quality/gtest_tool_confidence``.

#. **Canary suite (M1).** In every CI configuration that runs unit tests, a canary suite
   is built with the same toolchain, flags and GoogleTest version as the tests. It
   contains, for every GoogleTest/gMock feature in use, test cases that must fail and test
   cases that must pass, skip or be disabled. An independent checker (not GoogleTest)
   verifies the exit code, the presence of every listed test case in the XML report, the
   reported result of every test case and the ``RecordProperty()`` values. Any deviation
   fails CI.
#. **Test-run completeness check (M2).** After the unit tests, every ``test.xml`` is
   checked: the report is readable, contains test cases, the declared test count matches
   the reported test cases, no test case has an ``<error>``, every test listed by the
   binary (``--gtest_list_tests``) is present in the report, and skipped or disabled test
   cases that declare requirement links are reported.
#. **GoogleTest fault injection (M3).** Centrally, once per GoogleTest version and per
   toolchain change (not per module), faults are injected into the GoogleTest/gMock
   sources (Mull operators: arithmetic, comparison, boundary, logical, removed void call,
   replaced scalar call) and the canary suite with its checker is run against every
   faulty GoogleTest. Every undetected fault is classified (equivalent with respect to
   test verdicts and reports, or gap in the canary suite); gaps are closed by new
   canaries. The detection rate and the classification are archived with this report.
   This measures how well M1 detects real GoogleTest defects.

   Status: a first campaign with GoogleTest 1.17.0.bcr.2 (Clang 19, Mull 0.34.2) found
   1,710 mutants in the GoogleTest/gMock sources. Their execution against the canary
   checker is pending; until its result is archived, the detection claims of this
   report that rely on M3 are not substantiated.
#. **CI status check (M4).** A failing, crashing or timed-out test target fails the CI job;
   the job result is part of the release checklist.
#. **Single evaluated version (M5).** All modules use the GoogleTest version named in this
   report. A version change requires an update of this report and a canary run.

M1 runs in every module and configuration; M3 shows centrally that M1 is sensitive to
real GoogleTest defects, so a faulty GoogleTest cannot pass the canary suite unnoticed.
Per module, only M1 and M2 are required.

Mutation testing of the module's production code measures the strength of the module's
own tests (unit verification, ISO 26262-6), not the reliability of GoogleTest, and is
therefore not a measure of this report.

Report record
-------------

.. doc_tool:: GoogleTest
   :id: doc_tool__gtest
   :status: evaluated
   :version: 2
   :tool_version: 1.17.0.bcr.2
   :security_affected: YES
   :realizes: wp__tool_verification_report[version==1]
   :tags: tool_management, tools_testing_frameworks
   :post_template: tool_qualification_report

   Evaluates GoogleTest and gMock for executing C++ unit tests and reporting their
   results, including the requirement-verification properties used by docs-as-code.

Safety evaluation
-----------------

Each use case below records how S-CORE relies on GoogleTest. The potential malfunctions
are evaluated against the intended usage concept above.

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
      :detection_sufficient: YES
      :safety_measures: M1 canary suite per CI configuration; M3 GoogleTest fault injection (central, per version)
      :violates:
         tool_req__gtest_nonfatal_failure_reported,
         tool_req__gtest_fatal_failure_reported,
         tool_req__gtest_exception_failure_reported,
         tool_req__gtest_fixture_failure_reported
      :version: 1

      GoogleTest fails to detect an existing error: an assertion that is violated, an
      uncaught exception or a failure in ``SetUp()`` does not mark the test case as failed,
      or a fatal assertion does not stop the test.

      M1 executes a must-fail canary for every assertion kind in use in the same tool
      configuration as the tests; a canary reported as passed fails CI. M3 injects faults
      into the GoogleTest sources and shows that the canary suite detects them, so a
      GoogleTest defect of this kind cannot pass M1 unnoticed. Together the measures detect
      this malfunction with high confidence.

   .. potential_tool_malfunction:: Generated test instances are lost or merged
      :id: potential_tool_malfunction__gtest__gen_tests
      :safety_affected: YES
      :detection_sufficient: YES
      :safety_measures: M1 canary suite (per-instance results); M2 completeness check
      :violates: tool_req__gtest_generated_tests_reported
      :version: 1

      Value-parameterized or typed test instances are not generated, not executed, or
      their results are attributed to the wrong instance, so a failing instance is hidden.

      M1 contains parameterized and typed canaries in which only one instance fails and
      checks every instance's result. M2 compares the instances listed by each test binary
      with the report.

   .. potential_tool_malfunction:: Non-existing error is reported
      :id: potential_tool_malfunction__gtest__false_fail
      :safety_affected: NO
      :violates: tool_req__gtest_no_false_failure
      :version: 1

      GoogleTest reports a failure although the code under test is correct. The failure is
      visible and leads to analysis; it cannot hide an error. M1 must-pass canaries detect
      it as well.

.. tool_usecase:: Verify mock expectations
   :id: tool_usecase__gtest__mocks
   :belongs_to: doc_tool__gtest
   :version: 1

   A unit test uses gMock (``EXPECT_CALL``, ``StrictMock``, ``NiceMock``, ``InSequence``,
   cardinalities, argument matchers) to check the interaction of the code under test with
   its dependencies. A violated expectation must result in a failed test case.

   .. potential_tool_malfunction:: Violated mock expectation is not reported
      :id: potential_tool_malfunction__gtest__mock_pass
      :safety_affected: YES
      :detection_sufficient: YES
      :safety_measures: M1 canary suite per CI configuration; M3 GoogleTest fault injection (central, per version)
      :violates: tool_req__gtest_mock_expectation_verified
      :version: 1

      An unmet expectation, a call with unexpected arguments, too many calls, an
      uninteresting call on a ``StrictMock`` or a sequence violation is not reported.

      M1 contains a must-fail canary per kind of violation. M3 injects faults into the gMock
      sources (``gmock-spec-builders.cc`` and others) and shows that the canaries detect
      them.

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
      :detection_sufficient: YES
      :safety_measures: M1 canary suite per CI configuration incl. QNX; M3 GoogleTest fault injection (central, per version)
      :violates: tool_req__gtest_death_test_reported
      :version: 1

      A death test passes although the statement does not terminate, terminates with a
      different message, or exits with a different code. Death tests depend on the
      operating system (process creation), so the canary suite runs on every target OS.

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
      :safety_measures: M1 canary suite checks the exit code of the must-fail binary; M2 completeness check reads the per-test results
      :violates: tool_req__gtest_exit_code
      :version: 1

      The test binary exits with 0 although test cases failed, so Bazel and CI report the
      target as passed.

   .. potential_tool_malfunction:: Test result is missing from the report
      :id: potential_tool_malfunction__gtest__lost_res
      :safety_affected: YES
      :detection_sufficient: YES
      :safety_measures: M2 completeness check; M1 canary suite compares listed and reported test cases
      :violates: tool_req__gtest_report_complete
      :version: 1

      A test case is not executed, or its result is not written to the report (for
      example after a crash, with sharding or with filters), so a requirement appears
      verified by an incomplete run. A test calling ``exit(0)`` ends the binary with exit
      code 0 before the remaining tests run; GoogleTest leaves the file named by
      ``TEST_PREMATURE_EXIT_FILE`` behind and Bazel fails the target. The canary suite
      verifies this mechanism.

   .. potential_tool_malfunction:: Skipped or disabled test is reported as passed
      :id: potential_tool_malfunction__gtest__skip_pass
      :safety_affected: YES
      :detection_sufficient: YES
      :safety_measures: M1 canary suite (skipped and disabled canaries); M2 completeness check reports skipped/disabled tests with requirement links
      :violates: tool_req__gtest_skip_disabled_reported
      :version: 1

      docs-as-code treats a test case without ``<failure>``, ``<error>`` or ``<skipped>``
      element as passed. If GoogleTest omits the ``<skipped>`` element or the
      ``status="notrun"`` attribute, a test that verified nothing counts as verification
      evidence.

   .. potential_tool_malfunction:: Requirement properties are wrong in the report
      :id: potential_tool_malfunction__gtest__props
      :safety_affected: YES
      :detection_sufficient: YES
      :safety_measures: M1 canary suite checks RecordProperty values in the XML report; docs-as-code reports unresolved requirement references
      :violates: tool_req__gtest_record_property_reported
      :version: 1

      A ``RecordProperty()`` value (``FullyVerifies``, ``PartiallyVerifies``, ...) is lost,
      truncated or attached to the wrong test case, so the traceability shows a
      requirement as verified by the wrong test or not at all.

   .. potential_tool_malfunction:: Report cannot be written or input cannot be loaded
      :id: potential_tool_malfunction__gtest__io
      :safety_affected: YES
      :detection_sufficient: YES
      :safety_measures: M4 CI status check; M2 completeness check (missing or unreadable report)
      :violates: tool_req__gtest_report_complete
      :version: 1

      GoogleTest fails to load a provided file or to write the report. The test target
      fails or the report is missing or unreadable; both are visible.

Tool requirements
-----------------

The capabilities of GoogleTest that S-CORE relies on. The canary suite (M1) contains at
least one canary per requirement.

.. tool_req:: Non-fatal assertion failures are reported
   :id: tool_req__gtest_nonfatal_failure_reported
   :version: 1
   :satisfies: gd_req__verification_reporting[version==1]

   A violated ``EXPECT_*`` assertion (including ``EXPECT_THAT`` with matchers, floating
   point and C string comparisons and user-defined ``operator==``) and ``ADD_FAILURE()``
   shall mark the test case as failed.

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
   :version: 1
   :satisfies: gd_req__verification_reporting[version==1]

   A failure in a test fixture's ``SetUp()`` shall mark the test case as failed.

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
   :version: 1
   :satisfies:
      gd_req__verification_link_tests_cpp[version==1],
      gd_req__verification_reporting[version==1]

   Values recorded with ``RecordProperty()`` shall appear unchanged in the XML report at the
   test case that recorded them, and values recorded after a fatal failure shall not appear.

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

#. Run the canary suite (M1) in every CI configuration that runs unit tests, including
   QNX targets.
#. Run the test-run completeness check (M2) after the unit tests; treat its errors as
   test failures.
#. Before a GoogleTest version or toolchain change is used for verification evidence,
   repeat the GoogleTest fault-injection campaign (M3) centrally and archive its result.
#. Do not use ``GTEST_SKIP()`` or ``DISABLED_`` on test cases that declare
   ``FullyVerifies``/``PartiallyVerifies`` for a safety-related requirement.
#. Use the GoogleTest version named in this report (M5).
#. Do not set ``GTEST_FILTER``, ``--gtest_filter`` or ``--gtest_also_run_disabled_tests``
   in CI runs whose reports are used as verification evidence, and do not unset
   ``TEST_PREMATURE_EXIT_FILE`` (set by Bazel).
#. To detect unexpected calls, use ``StrictMock`` or ``EXPECT_CALL(...).Times(0)``. A
   default ("naggy") mock only prints a warning for an uninteresting call and does not
   fail the test; the canary suite pins this behaviour.

Security evaluation
-------------------

.. list-table:: Security evaluation
   :header-rows: 1

   * - Threat identification
     - Use case description
     - Threats
     - Impact on security?
     - Impact security measures available?
     - Impact security detection sufficient?
   * - 1
     - TBD
     - TBD
     - TBD
     - TBD
     - TBD

Result
~~~~~~

With the intended usage concept (measures M1 to M5), every safety-relevant potential
malfunction is detected with sufficient confidence. GoogleTest does not require
qualification; the confidence level is derived automatically from the use cases and
malfunctions above.
