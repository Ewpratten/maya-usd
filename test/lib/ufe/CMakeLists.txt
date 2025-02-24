# unit test scripts.  Note that testTRSBase.py is not a test case, but rather
# a module providing a base class for other tests.
set(TEST_SCRIPT_FILES
    testDeleteCmd.py
    testMatrices.py
    testMayaPickwalk.py
    testPythonWrappers.py
    testSelection.py
    testSelectionByArray.py
    testUfePythonImport.py
    testAttributeBlock.py
    testBlockedLayerEdit.py
)

set(TEST_SUPPORT_FILES
    testTRSBase.py
)

if(CMAKE_UFE_V2_FEATURES_AVAILABLE)
    list(APPEND TEST_SCRIPT_FILES
        testAttribute.py
        testAttributes.py
        testChildFilter.py
        testComboCmd.py
        testContextOps.py
        testDuplicateCmd.py
        testGroupCmd.py
        testMoveCmd.py
        testObject3d.py
        testRename.py
        testParentCmd.py
        testPointInstances.py
        testReorderCmd.py
        testRotateCmd.py
        testRotatePivot.py
        testScaleCmd.py
        testSceneItem.py
        testTransform3dChainOfResponsibility.py
        testTransform3dTranslate.py
        testUIInfoHandler.py
        testObservableScene.py
    )
endif()

if(CMAKE_UFE_V3_FEATURES_AVAILABLE)
    list(APPEND TEST_SCRIPT_FILES
        testUngroupCmd.py
    )
endif()

if (MAYA_API_VERSION VERSION_GREATER_EQUAL 20220100)
    list(APPEND TEST_SCRIPT_FILES
        testCamera.py
    )
endif()

if(MAYA_MRENDERITEM_UFE_IDENTIFIER_SUPPORT)
    list(APPEND TEST_SCRIPT_FILES
        testSetsCmd.py
    )
endif()

foreach(script ${TEST_SCRIPT_FILES})
    mayaUsd_get_unittest_target(target ${script})
    mayaUsd_add_test(${target}
        WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
        PYTHON_MODULE ${target}
        ENV
            "MAYA_PLUG_IN_PATH=${CMAKE_CURRENT_SOURCE_DIR}/ufeTestPlugins"
            "UFE_PREVIEW_VERSION_NUM=${UFE_PREVIEW_VERSION_NUM}"
            "LD_LIBRARY_PATH=${ADDITIONAL_LD_LIBRARY_PATH}"
    )

    # Add a ctest label to these tests for easy filtering.
    set_property(TEST ${target} APPEND PROPERTY LABELS ufe)
endforeach()
