#######################################################################################
# The MIT License

# Copyright (c) 2014       Hannes Schulz, University of Bonn  <schulz@ais.uni-bonn.de>
# Copyright (c) 2013       Benedikt Waldvogel, University of Bonn <mail@bwaldvogel.de>
# Copyright (c) 2008-2009  Sebastian Nowozin                       <nowozin@gmail.com>

# Permission is hereby granted, free of charge, to any person obtaining a copy
# of this software and associated documentation files (the "Software"), to deal
# in the Software without restriction, including without limitation the rights
# to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
# copies of the Software, and to permit persons to whom the Software is
# furnished to do so, subject to the following conditions:
# 
# The above copyright notice and this permission notice shall be included in all
# copies or substantial portions of the Software.
# 
# THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
# IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
# FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
# AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
# LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
# OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
# SOFTWARE.
#######################################################################################
# http://stackoverflow.com/questions/1435953/how-can-i-pass-git-sha1-to-compiler-as-definition-using-cmake/4318642#4318642

configure_file("${CMAKE_CURRENT_SOURCE_DIR}/version.cpp.in" "${CMAKE_CURRENT_BINARY_DIR}/version.cpp" @ONLY)

IF(MDBQ_FOUND)
	SET (MDBQ_FILES hyperopt.cpp)
ELSE()
	SET (MDBQ_FILES )
	SET (MDBQ_LIBRARIES )
ENDIF()

CUDA_ADD_LIBRARY(curfil  random_tree_image_gpu.cu random_tree.cpp image.cpp utils.cpp ndarray_ops.cpp random_tree_image.cpp random_forest_image.cpp import.cpp export.cpp predict.cpp ndarray_ops.cpp train.cpp ${MDBQ_FILES} "${CMAKE_CURRENT_BINARY_DIR}/version.cpp")

MACRO(VIGRA_FIND_PACKAGE package)
    # parse the args
    set(COMPONENTS)
    set(NAMES)
    set(VERSION_SPEC)
    set(v VERSION_SPEC)
    foreach(i ${ARGN})
        if(${i} MATCHES "^COMPONENTS$")
            set(v COMPONENTS)
        elseif(${i} MATCHES "^NAMES$")
            set(v NAMES)
        else()
            set(${v} ${${v}} ${i})
        endif()
    endforeach(i)

    SET(${package}_NAMES ${NAMES} ${${package}_NAMES})

    IF(DEFINED COMPONENTS)
        SET(COMPONENTS COMPONENTS ${COMPONENTS})
    ENDIF()
    
    if(VERSION_SPEC)
        set(VERSION_MESSAGE " (at least version ${VERSION_SPEC})")
    else()
        set(VERSION_MESSAGE)
    endif()
    
    MESSAGE(STATUS "Searching for ${package}${VERSION_MESSAGE}")
    
    foreach(path ${DEPENDENCY_SEARCH_PREFIX})
        if(NOT ${package}_FOUND)
            IF(${package} STREQUAL "Boost")
                IF(EXISTS "${path}/boost/config.hpp")
                    SET(BOOST_INCLUDEDIR ${path}) # boost's default include path
                ELSE()
                    SET(BOOST_INCLUDEDIR ${path}/include) # standard include path
                ENDIF()
                IF(EXISTS "${path}/stage/lib")
                    SET(BOOST_LIBRARYDIR ${path}/stage/lib) # boost's default library path
                ELSE()
                    SET(BOOST_LIBRARYDIR ${path}/lib) # standard library path
                ENDIF()
            ELSE()
                SET(CMAKE_INCLUDE_PATH ${path}/include)
                SET(CMAKE_LIBRARY_PATH ${path}/lib)
            ENDIF()
            # SET(${package}_INCLUDE_DIR ${package}_INCLUDE_DIR-NOTFOUND)
            # SET(${package}_LIBRARIES ${package}_LIBRARIES-NOTFOUND)
            # SET(${package}_LIBRARY ${package}_LIBRARY-NOTFOUND)
            MESSAGE(STATUS "   in prefix ${path}")
            FIND_PACKAGE(${package} ${VERSION_SPEC} ${COMPONENTS})
            IF(${package} STREQUAL "Boost")
                SET(BOOST_INCLUDEDIR)
                SET(BOOST_LIBRARYDIR)
            ELSE()
                SET(CMAKE_INCLUDE_PATH)
                SET(CMAKE_LIBRARY_PATH)
            ENDIF()
        endif()
    endforeach(path)
   
    # search the package in the default locations if not found 
    # in the DEPENDENCY_SEARCH_PREFIX 
    if(NOT ${package}_FOUND)
        MESSAGE(STATUS "   in default locations")
        FIND_PACKAGE(${package} ${VERSION_SPEC} ${COMPONENTS})
    endif()
    
ENDMACRO(VIGRA_FIND_PACKAGE)    


VIGRA_FIND_PACKAGE(ZLIB)
VIGRA_FIND_PACKAGE(TIFF NAMES libtiff_i libtiff) # prefer DLL on Windows
VIGRA_FIND_PACKAGE(JPEG NAMES libjpeg)
VIGRA_FIND_PACKAGE(PNG)
VIGRA_FIND_PACKAGE(FFTW3 NAMES libfftw3-3 libfftw-3.3)
VIGRA_FIND_PACKAGE(FFTW3F NAMES libfftw3f-3 libfftwf-3.3)


IF(WITH_OPENEXR)
    VIGRA_FIND_PACKAGE(OpenEXR)
ENDIF()

IF(WITH_HDF5)
    VIGRA_FIND_PACKAGE(HDF5)
ENDIF()

IF(WITH_LEMON)
    VIGRA_FIND_PACKAGE(LEMON)
ENDIF()


TARGET_LINK_LIBRARIES(curfil ndarray ${CUDA_LIBRARIES} ${VIGRA_LIBRARIES} ${Boost_LIBRARIES} 
${MDBQ_LIBRARIES} ${JPEG_LIBRARIES} ${PNG_LIBRARIES}  ${TIFF_LIBRARIES}  ${ZLIB_LIBRARIES} pthread)

INSTALL(TARGETS curfil
	DESTINATION "lib"
)

INSTALL(FILES random_tree.h random_tree_image.h random_forest_image.h image.h score.h random_tree_image_gpu.h predict.h import.h export.h utils.h
	DESTINATION "include/curfil"
)

ADD_EXECUTABLE(curfil_train train_main.cpp)
TARGET_LINK_LIBRARIES(curfil_train curfil)

ADD_EXECUTABLE(curfil_predict predict_main.cpp)
TARGET_LINK_LIBRARIES(curfil_predict curfil)

INSTALL(TARGETS curfil_train curfil_predict
    DESTINATION "bin"
)

IF(MDBQ_FOUND)
	ADD_EXECUTABLE(curfil_hyperopt hyperopt_main.cpp)
	TARGET_LINK_LIBRARIES(curfil_hyperopt curfil)
ENDIF()
