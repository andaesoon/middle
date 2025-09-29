<!DOCTYPE html>
<html lang="ko">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>중학교 배정 거리 계산 웹 앱</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <!-- 네이버 API는 웹 서비스 API이므로 별도의 JS 라이브러리 로딩은 필요하지 않습니다. -->
</head>
<body class="bg-gray-50 min-h-screen p-4 md:p-8 font-sans">

    <!-- Tailwind CSS 설정: 'Inter' 폰트 사용 및 커스텀 색상 정의 (선택 사항) -->
    <script>
        tailwind.config = {
            theme: {
                extend: {
                    fontFamily: {
                        sans: ['Inter', 'sans-serif'],
                    },
                    colors: {
                        'primary-blue': '#1D4ED8',
                        'secondary-gray': '#4B5563',
                    }
                }
            }
        }
    </script>

    <div id="app" class="max-w-6xl mx-auto">
        <header class="text-center mb-8">
            <h1 class="text-4xl font-extrabold text-primary-blue mb-2">중학교 배정 거리 분석 시스템 (Naver API)</h1>
            <p class="text-secondary-gray text-lg">실거주지 및 중학교 목록을 입력하여 네이버 지도를 기반으로 최단 도로 거리를 계산합니다.</p>
        </header>

        <main class="bg-white p-6 md:p-10 shadow-xl rounded-2xl">
            <!-- 안내 및 API 키 설정 영역 -->
            <div class="mb-8 border-b pb-4">
                <h2 class="text-2xl font-bold text-gray-700 mb-4">📍 데이터 입력 및 네이버 API 설정</h2>
                <p class="text-sm text-red-600 mb-4 p-3 bg-red-50 rounded-lg">
                    ⚠️ **중요 안내:** 현재 앱은 **실제 API 호출 모드**로 작동합니다. 계산이 실패할 경우, 네이버 개발자 센터에서 **Client ID 및 Client Secret의 유효성**과 **서비스 URL 제한 해제**가 필수입니다.
                </p>

                <div class="grid grid-cols-1 md:grid-cols-2 gap-4">
                    <div>
                        <label for="naverClientId" class="block text-sm font-medium text-gray-700">Naver Maps API Client ID</label>
                        <input type="text" id="naverClientId" value="6wf0ftkial" class="mt-1 block w-full border border-gray-300 rounded-lg shadow-sm p-3 focus:ring-primary-blue focus:border-primary-blue" placeholder="Client ID를 입력하세요">
                    </div>
                     <div>
                        <label for="naverClientSecret" class="block text-sm font-medium text-gray-700">Naver Maps API Client Secret</label>
                        <input type="password" id="naverClientSecret" value="VT7zKZin6izD3Pv1bQYDbEqo39ApQijmbw9PcTsl" class="mt-1 block w-full border border-gray-300 rounded-lg shadow-sm p-3 focus:ring-primary-blue focus:border-primary-blue" placeholder="Client Secret을 입력하세요">
                    </div>
                </div>
            </div>

            <!-- 데이터 입력 영역 (파일 업로드 + 텍스트 입력) -->
            <div class="grid grid-cols-1 md:grid-cols-2 gap-6 mb-8">
                <!-- 실거주지 파일 업로드 영역 -->
                <div>
                    <label class="block text-lg font-bold text-gray-700 mb-2">🏠 실거주지 목록 (CSV 파일 업로드)</label>
                    <p class="text-sm text-gray-500 mb-2">CSV 파일 업로드. 형식: **성명, 주소** (각 줄에 하나씩)</p>
                    <input type="file" id="residenceFileInput" accept=".csv" class="block w-full text-sm text-gray-500 file:mr-4 file:py-2 file:px-4 file:rounded-full file:border-0 file:text-sm file:font-semibold file:bg-primary-blue/10 file:text-primary-blue hover:file:bg-primary-blue/20"/>
                    
                    <!-- 로드된 데이터 미리보기 및 정보 표시 -->
                    <div id="residenceDataPreview" class="mt-3 p-3 text-sm bg-gray-100 border border-gray-300 rounded-lg h-32 overflow-y-auto text-gray-700 hidden">
                        <p class="font-semibold mb-1">불러온 데이터 미리보기 (최대 5줄):</p>
                        <pre id="residencePreviewContent" class="whitespace-pre-wrap text-xs"></pre>
                        <span id="residenceLoadedDataCount" class="text-xs text-green-600 mt-1 block"></span>
                    </div>
                    <!-- 실제 계산에 사용될 원본 데이터 저장용 hidden input -->
                    <input type="hidden" id="residenceRawData" data-initial-value="김관우, 서울특별시 강남구 테헤란로 132
학생2, 서울특별시 서초구 효령로 188
학생3, 서울특별시 송파구 올림픽로 300" value="김관우, 서울특별시 강남구 테헤란로 132
학생2, 서울특별시 서초구 효령로 188
학생3, 서울특별시 송파구 올림픽로 300"> 
                    <p id="residenceInitialDataWarning" class="text-xs text-gray-500 mt-2">파일 업로드 전에는 기본 데이터가 사용됩니다.</p>
                </div>

                <!-- 중학교 목록 텍스트 영역 (단순 입력으로 복원) -->
                <div>
                    <label for="schoolInput" class="block text-lg font-bold text-gray-700 mb-2">🏫 중학교 목록 (CSV 텍스트 입력)</label>
                    <p class="text-sm text-gray-500 mb-2">형식: 학교명, 주소 (각 줄에 하나씩)</p>
                    <textarea id="schoolInput" rows="11" class="w-full border border-gray-300 rounded-lg p-3 text-sm focus:ring-primary-blue focus:border-primary-blue" placeholder="예시:&#10;강남중학교, 서울특별시 강남구 삼성로 508&#10;서초중학교, 서울특별시 서초구 효령로 188&#10;송파중학교, 서울특별시 송파구 백제고분로 402">강남중학교, 서울특별시 강남구 삼성로 508
서초중학교, 서울특별시 서초구 서초대로 398
송파중학교, 서울특별시 송파구 백제고분로 402</textarea>
                </div>
            </div>

            <!-- 실행 버튼 -->
            <button id="calculateButton" class="w-full py-4 bg-primary-blue text-white font-bold text-xl rounded-xl shadow-lg hover:bg-blue-700 transition duration-300 ease-in-out disabled:opacity-50 flex items-center justify-center">
                <svg id="loadingSpinner" class="animate-spin -ml-1 mr-3 h-5 w-5 text-white hidden" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24">
                    <circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4"></circle>
                    <path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"></path>
                </svg>
                거리 계산 시작
            </button>

            <!-- 결과 출력 영역 -->
            <div id="resultsArea" class="mt-10 pt-6 border-t hidden">
                <h2 class="text-2xl font-bold text-gray-700 mb-4">📊 계산 결과 (도로 최단 경로)</h2>
                <div id="resultsTableContainer" class="overflow-x-auto">
                    <!-- 결과 테이블이 여기에 삽입됩니다 -->
                </div>
                <div id="errorMessages" class="mt-4 p-4 bg-yellow-100 text-yellow-800 rounded-lg hidden"></div>
                
                <h3 class="text-xl font-semibold text-gray-700 mt-8 mb-4">최적 배정 결과 (최단 거리 기준)</h3>
                <div id="optimalAssignments" class="grid grid-cols-1 md:grid-cols-3 gap-4">
                    <!-- 최적 배정 결과가 여기에 삽입됩니다 -->
                </div>
            </div>
        </main>
    </div>

    <script>
        // 상수 및 전역 변수 설정
        const RESULTS_LIMIT = 5; 
        const MOCK_MODE = false; // 실제 API 호출 모드로 전환했습니다.

        // --- 파일 처리 로직 (이전 버전과 동일) ---

        function handleFileUpload(event, prefix) {
            const file = event.target.files[0];
            const previewDiv = document.getElementById(`${prefix}DataPreview`);
            const previewContent = document.getElementById(`${prefix}PreviewContent`);
            const loadedDataCount = document.getElementById(`${prefix}LoadedDataCount`);
            const rawDataInput = document.getElementById(`${prefix}RawData`);
            const initialDataWarning = document.getElementById(`${prefix}InitialDataWarning`);

            if (!file) {
                previewDiv.classList.add('hidden');
                const initialValue = rawDataInput.getAttribute('data-initial-value') || "";
                rawDataInput.value = initialValue;
                initialDataWarning.classList.remove('hidden');
                return;
            }

            const reader = new FileReader();
            reader.onload = (e) => {
                const fileContent = e.target.result;
                
                rawDataInput.value = fileContent;
                initialDataWarning.classList.add('hidden');
                
                const lines = fileContent.split('\n').filter(line => line.trim().length > 0);

                const validLines = lines.filter(line => {
                    const parts = line.split(',').map(item => item.trim());
                    return parts.length >= 2 && parts[1] && parts[1].trim().length > 0;
                });

                const previewText = validLines.slice(0, 5).map(line => line.length > 50 ? line.substring(0, 50) + '...' : line).join('\n');
                previewContent.textContent = previewText;
                loadedDataCount.textContent = `✅ 총 ${validLines.length}개의 유효한 주소 데이터가 로드되었습니다.`;
                
                if (validLines.length === 0) {
                     loadedDataCount.textContent = `❌ 유효한 주소 데이터가 없습니다. CSV 형식을 확인해주세요 (예: 성명, 주소).`;
                     loadedDataCount.classList.remove('text-green-600');
                     loadedDataCount.classList.add('text-red-600');
                } else {
                     loadedDataCount.classList.remove('text-red-600');
                     loadedDataCount.classList.add('text-green-600');
                }
                
                previewDiv.classList.remove('hidden');

            };
            
            reader.readAsText(file, 'UTF-8');
        }

        /**
         * CSV 데이터를 파싱하여 객체 배열로 변환합니다.
         */
        function parseInput(rawData) {
            if (!rawData) return [];
            
            const lines = rawData.split('\n').filter(line => line.trim().length > 0);

            const parsedData = lines
                .map(line => line.split(',').map(item => item.trim()))
                .filter(parts => {
                    if (parts.length < 2) return false;
                    
                    const name = parts[0].trim();
                    const address = parts[1].trim();
                    
                    return name.length > 0 && address.length > 0;
                })
                .map(parts => ({
                    id: crypto.randomUUID(),
                    name: parts[0], 
                    address: parts[1]
                }));
                
            return parsedData;
        }


        // --- 네이버 API 호출을 위한 재시도 로직 ---

        async function fetchWithRetry(url, options, maxRetries = 3) {
            for (let i = 0; i < maxRetries; i++) {
                try {
                    const response = await fetch(url, options);
                    
                    // 네이버 API는 인증 실패 시 401/403, 기타 오류 시 4xx/5xx를 반환할 수 있음
                    if (!response.ok) {
                        const errorBody = await response.json().catch(() => ({}));
                        if (response.status === 401 || response.status === 403) {
                            throw new Error(`API 인증 실패 (401/403). Client ID 또는 Client Secret을 확인하세요.`);
                        }
                        
                        // 네이버 API의 일반적인 오류 메시지 구조를 따름
                        const errorMessage = errorBody.errorMessage || `HTTP 오류: ${response.status}`;
                        throw new Error(`네이버 API 호출 오류: ${errorMessage}`);
                    }
                    
                    return await response.json();

                } catch (error) {
                    // TypeError (Failed to fetch) 발생 시, 외부 네트워크/CORS 문제로 판단
                    if (error instanceof TypeError && error.message.includes('Failed to fetch')) {
                        throw new Error("네트워크 연결 실패 또는 CORS 보안 정책에 의한 요청 차단 오류입니다. 네이버 API 설정을 확인하세요.");
                    }
                    if (i === maxRetries - 1) throw error; 
                    await new Promise(resolve => setTimeout(resolve, Math.pow(2, i) * 1000)); 
                }
            }
        }

        // --- 네이버 API 연동 로직 ---

        /**
         * 주소 목록을 지리적 좌표로 변환합니다 (Naver Geocoding API).
         */
        async function geocodeAddresses(locations, clientId, clientSecret) {
            if (MOCK_MODE) return mockGeocodeAddresses(locations);

            const promises = locations.map(async (loc) => {
                const GEOCODE_URL = "https://naveropenapi.apigw.ntruss.com/map-geocode/v2/geocode";
                const url = `${GEOCODE_URL}?query=${encodeURIComponent(loc.address)}`;
                
                try {
                    const data = await fetchWithRetry(url, {
                        method: 'GET',
                        headers: {
                            'X-NCP-APIGW-API-KEY-ID': clientId,
                            'X-NCP-APIGW-API-KEY': clientSecret
                        }
                    });
                    
                    if (data.addresses && data.addresses.length > 0) {
                        const firstAddress = data.addresses[0];
                        return { 
                            ...loc, 
                            lat: parseFloat(firstAddress.y), 
                            lng: parseFloat(firstAddress.x),
                            apiStatus: 'OK' 
                        };
                    } else {
                        console.warn(`Geocoding 실패 - 주소: ${loc.address}. 결과 없음.`);
                        return { ...loc, lat: null, lng: null, apiStatus: 'ZERO_RESULTS' };
                    }

                } catch (error) {
                    console.error(`Geocoding API 호출 오류: ${loc.address}`, error);
                    return { ...loc, lat: null, lng: null, apiStatus: error.message || 'FETCH_ERROR', errorMessage: error.message };
                }
            });
            
            return Promise.all(promises); 
        }

        /**
         * 모든 거주지-학교 쌍에 대한 도로 거리를 계산합니다 (Naver Driving Direction API).
         * 네이버 API는 다대다 계산을 지원하지 않아, 개별 호출을 수행합니다.
         */
        async function calculateDistances(residences, schools, clientId, clientSecret) {
            if (MOCK_MODE) return mockCalculateDistances(residences, schools);

            const DISTANCE_URL = "https://naveropenapi.apigw.ntruss.com/map-direction/v1/driving";
            const allResults = [];

            // 모든 거주지-학교 쌍에 대해 API 호출
            for (const residence of residences) {
                for (const school of schools) {
                    // 네이버는 [lng,lat] 순서 (경도, 위도)
                    const origin = `${residence.lng},${residence.lat}`;
                    const destination = `${school.lng},${school.lat}`;

                    const url = `${DISTANCE_URL}?start=${origin}&goal=${destination}&option=trafast`; // trafast는 빠르게 계산

                    try {
                        const data = await fetchWithRetry(url, {
                            method: 'GET',
                            headers: {
                                'X-NCP-APIGW-API-KEY-ID': clientId,
                                'X-NCP-APIGW-API-KEY': clientSecret
                            }
                        });

                        const path = data.route?.trafast?.[0]?.summary;

                        if (path) {
                            allResults.push({
                                residenceId: residence.id,
                                residenceName: residence.name,
                                schoolId: school.id,
                                schoolName: school.name,
                                // 네이버 API는 meters와 seconds를 반환
                                distanceMeters: path.distance, 
                                distanceText: (path.distance / 1000).toFixed(2) + ' km',     
                                durationText: formatDuration(path.duration),     
                            });
                        } else {
                            // 경로를 찾지 못한 경우
                            console.warn(`거리 계산 실패 (거주지: ${residence.name}, 학교: ${school.name}): 경로 없음`);
                        }
                    } catch (error) {
                        console.error(`Distance API 호출 오류: ${residence.name} -> ${school.name}`, error);
                    }
                }
            }

            return allResults;
        }

        /**
         * 밀리초 단위의 시간을 텍스트로 변환 (예: 360000ms -> 6분)
         */
        function formatDuration(ms) {
            const seconds = Math.floor(ms / 1000);
            const minutes = Math.floor(seconds / 60);
            const hours = Math.floor(minutes / 60);

            if (hours > 0) {
                return `${hours}시간 ${minutes % 60}분`;
            } else if (minutes > 0) {
                return `${minutes}분`;
            } else {
                return `${seconds}초`;
            }
        }


        // --- MOCK Functions (모의 데이터 반환) ---

        /**
         * (MOCK) 주소 목록을 가짜 좌표로 변환합니다.
         */
        async function mockGeocodeAddresses(locations) {
            await new Promise(resolve => setTimeout(resolve, 300)); 
            return locations.map((loc, index) => ({
                ...loc,
                lat: 37.5000 + (index % 3) * 0.005,
                lng: 127.0300 + (index % 3) * 0.005,
                apiStatus: 'OK' 
            }));
        }

        /**
         * (MOCK) 모든 거주지-학교 쌍에 대한 모의 거리 데이터를 생성합니다.
         */
        async function mockCalculateDistances(residences, schools) {
            await new Promise(resolve => setTimeout(resolve, 800)); 

            const results = [];
            const BASE_DIST = 1.5; 

            residences.forEach((residence, rIndex) => {
                schools.forEach((school, sIndex) => {
                    const km = (BASE_DIST + Math.abs(rIndex - sIndex) * 0.5 + Math.random() * 0.3).toFixed(2);
                    const meters = Math.round(km * 1000);
                    const durationMins = Math.round(meters / 60);

                    results.push({
                        residenceId: residence.id,
                        residenceName: residence.name,
                        schoolId: school.id,
                        schoolName: school.name,
                        distanceMeters: meters,
                        distanceText: `${km} km`,
                        durationText: `${durationMins} 분`,
                    });
                });
            });

            return results;
        }


        /**
         * 메인 계산 로직을 실행합니다.
         */
        async function runCalculation() {
            const calculateButton = document.getElementById('calculateButton');
            const resultsArea = document.getElementById('resultsArea');
            const errorMessages = document.getElementById('errorMessages');
            
            const clientId = document.getElementById('naverClientId').value;
            const clientSecret = document.getElementById('naverClientSecret').value;

            const residenceRawData = document.getElementById('residenceRawData');
            const residenceDataString = residenceRawData.value;
            const schoolDataString = document.getElementById('schoolInput').value; 

            // UI 상태 업데이트: 비활성화 및 스피너 표시
            calculateButton.disabled = true;
            calculateButton.innerHTML = '<svg class="animate-spin -ml-1 mr-3 h-5 w-5 text-white" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24"><circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4"></circle><path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"></path></svg> 거리 계산 중...';
            resultsArea.classList.add('hidden');
            errorMessages.classList.add('hidden');
            errorMessages.innerHTML = '';

            // API Key 유효성 검사 (MOCK 모드에서는 건너뜀)
            if (!MOCK_MODE && (!clientId || !clientSecret)) {
                errorMessages.innerHTML = `<strong>오류:</strong> 네이버 API Client ID와 Client Secret을 모두 입력해주세요.`;
                errorMessages.classList.remove('hidden');
                calculateButton.disabled = false;
                calculateButton.innerHTML = '거리 계산 시작';
                return;
            }

            try {
                // 1. 입력 데이터 파싱 
                const residences = parseInput(residenceDataString);
                const schools = parseInput(schoolDataString); 

                if (residences.length === 0) {
                    throw new Error("거주지 목록에 유효한 주소 데이터가 없습니다. CSV 형식을 확인해주세요 (예: 성명, 주소).");
                }
                if (schools.length === 0) {
                    throw new Error("학교 목록에 유효한 주소 데이터가 없습니다. CSV 형식을 확인해주세요 (예: 학교명, 주소).");
                }

                // 2. 주소 변환 (Geocoding)
                const geocodedResidencesWithStatus = await geocodeAddresses(residences, clientId, clientSecret);
                const geocodedSchoolsWithStatus = await geocodeAddresses(schools, clientId, clientSecret);

                // 좌표를 성공적으로 얻은 항목만 필터링
                const geocodedResidences = geocodedResidencesWithStatus.filter(r => r.apiStatus === 'OK');
                const geocodedSchools = geocodedSchoolsWithStatus.filter(r => r.apiStatus === 'OK');

                // 네이버 API 오류 진단 (Client Secret 오류가 가장 흔함)
                if (geocodedResidences.length === 0 && !MOCK_MODE) {
                    const errorMessages = geocodedResidencesWithStatus.map(r => r.errorMessage).filter(m => m && m.includes('인증 실패'));
                    if (errorMessages.length > 0) {
                         throw new Error(`API 오류: Geocoding 요청이 거부되었습니다. Client ID와 Client Secret의 유효성을 확인해주세요.`);
                    }
                    throw new Error("모든 거주지 주소에 대한 지리적 좌표 변환에 실패했습니다. 주소가 정확한지 확인해 주세요.");
                }
                
                 if (geocodedSchools.length === 0 && !MOCK_MODE) {
                    const errorMessages = geocodedSchoolsWithStatus.map(r => r.errorMessage).filter(m => m && m.includes('인증 실패'));
                    if (errorMessages.length > 0) {
                         throw new Error(`API 오류: Geocoding 요청이 거부되었습니다. Client ID와 Client Secret의 유효성을 확인해주세요.`);
                    }
                    throw new Error("모든 학교 주소에 대한 지리적 좌표 변환에 실패했습니다. 주소가 정확한지 확인해 주세요.");
                }


                // 3. 거리 계산 (Driving Direction)
                const allResults = await calculateDistances(geocodedResidences, geocodedSchools, clientId, clientSecret);

                if (allResults.length === 0) {
                     throw new Error("모든 거주지-학교 쌍에 대한 경로 계산에 실패했습니다. 주소 정확도 및 API 설정을 확인해주세요.");
                }

                // 4. 결과 렌더링
                renderResults(allResults, geocodedResidences, geocodedSchools);

                // 5. UI 상태 업데이트
                resultsArea.classList.remove('hidden');

            } catch (error) {
                console.error("거리 계산 오류:", error);
                
                let displayMessage;
                if (error.message.includes('네트워크 연결 실패')) {
                    displayMessage = `<strong>연결 오류:</strong> ${error.message}. **CORS 문제** 해결을 위해 네이버 개발자 센터에서 앱 설정을 확인해 주세요.`;
                } else if (error.message.includes('인증 실패')) {
                    displayMessage = `<strong>API 인증 오류:</strong> ${error.message}. 네이버 **Client ID와 Client Secret**이 유효한지 확인해주세요.`;
                } else {
                     displayMessage = `<strong>오류 발생:</strong> ${error.message}`;
                }


                errorMessages.innerHTML = displayMessage;
                errorMessages.classList.remove('hidden');
            } finally {
                // UI 상태 초기화
                calculateButton.disabled = false;
                calculateButton.innerHTML = '거리 계산 시작';
            }
        }

        // --- 결과 렌더링 로직 (변경 없음) ---

        /**
         * 계산된 결과를 화면에 표시합니다.
         */
        function renderResults(allResults, residences) {
            const tableContainer = document.getElementById('resultsTableContainer');
            const optimalAssignmentsDiv = document.getElementById('optimalAssignments');
            tableContainer.innerHTML = '';
            optimalAssignmentsDiv.innerHTML = '';

            // 1. 학생별로 결과 그룹화 및 정렬
            const resultsByResidence = residences.map(residence => {
                const studentResults = allResults
                    .filter(r => r.residenceId === residence.id)
                    .sort((a, b) => a.distanceMeters - b.distanceMeters); // 거리 오름차순 정렬

                return {
                    residenceName: residence.name,
                    id: residence.id,
                    results: studentResults
                };
            });

            // 2. 전체 결과를 담을 HTML 문자열 생성 (테이블)
            let tableHTML = `
                <table class="min-w-full divide-y divide-gray-200 rounded-xl shadow-md">
                    <thead class="bg-primary-blue/90">
                        <tr>
                            <th class="px-6 py-3 text-left text-xs font-medium text-white uppercase tracking-wider sticky left-0 z-10">실거주지 (성명)</th>
                            <th class="px-6 py-3 text-left text-xs font-medium text-white uppercase tracking-wider">학교명</th>
                            <th class="px-6 py-3 text-right text-xs font-medium text-white uppercase tracking-wider">거리</th>
                            <th class="px-6 py-3 text-right text-xs font-medium text-white uppercase tracking-wider">예상 시간</th>
                        </tr>
                    </thead>
                    <tbody class="bg-white divide-y divide-gray-200">
            `;

            // 3. 테이블 행 추가
            const bestAssignments = []; 
            let rowCount = 0;

            resultsByResidence.forEach((residenceData, index) => {
                const totalResults = residenceData.results.length;
                let isFirstRow = true;

                // 최단 거리 학교 (최적 배정) 저장
                if (totalResults > 0) {
                    bestAssignments.push(residenceData.results[0]);
                }
                
                // 상위 RESULTS_LIMIT 개 결과만 표시
                const displayResults = residenceData.results.slice(0, RESULTS_LIMIT);
                
                displayResults.forEach(result => {
                    const isBest = isFirstRow; 
                    const rowClass = isBest ? 'bg-blue-50 hover:bg-blue-100' : 'hover:bg-gray-50';
                    const nameClass = isBest ? 'font-bold text-primary-blue sticky left-0 z-10' : 'text-gray-900 sticky left-0 z-10';
                    
                    tableHTML += `
                        <tr class="${rowClass}">
                            ${isFirstRow ? 
                                `<td rowspan="${displayResults.length}" class="px-6 py-4 whitespace-nowrap text-sm ${nameClass} align-top border-r border-gray-200 bg-white/90">
                                    ${residenceData.residenceName}
                                </td>` : ''
                            }
                            <td class="px-6 py-4 whitespace-nowrap text-sm ${isBest ? 'font-semibold text-primary-blue' : 'text-gray-900'}">${result.schoolName}</td>
                            <td class="px-6 py-4 whitespace-nowrap text-sm text-right ${isBest ? 'font-bold text-lg text-green-600' : 'text-gray-700'}">${result.distanceText}</td>
                            <td class="px-6 py-4 whitespace-nowrap text-sm text-right text-gray-700">${result.durationText}</td>
                        </tr>
                    `;
                    isFirstRow = false;
                    rowCount++;
                });

                // 학생별 구분선
                if (index < resultsByResidence.length - 1) {
                    tableHTML += `<tr><td colspan="4" class="h-2 bg-gray-100"></td></tr>`;
                }
            });

            tableHTML += '</tbody></table>';
            tableContainer.innerHTML = tableHTML;
            
            // 4. 최적 배정 결과 카드 렌더링
            bestAssignments.forEach(assignment => {
                optimalAssignmentsDiv.innerHTML += `
                    <div class="p-5 bg-green-50 border border-green-200 rounded-xl shadow-lg">
                        <p class="text-sm font-semibold text-green-700 mb-1">최단 거리 학교</p>
                        <h4 class="text-xl font-bold text-gray-800 mb-2">${assignment.residenceName}</h4>
                        <p class="text-lg text-primary-blue">
                            <span class="font-semibold">${assignment.schoolName}</span>
                        </p>
                        <p class="text-3xl font-extrabold text-green-600 mt-2">${assignment.distanceText}</p>
                        <p class="text-sm text-gray-500">운전 예상 시간: ${assignment.durationText}</p>
                    </div>
                `;
            });

        }

        // --- 이벤트 리스너 설정 ---

        document.addEventListener('DOMContentLoaded', () => {
            const calculateButton = document.getElementById('calculateButton');
            const residenceFileInput = document.getElementById('residenceFileInput');
            
            calculateButton.addEventListener('click', runCalculation);
            
            // 거주지 파일 변경 이벤트
            residenceFileInput.addEventListener('change', (e) => handleFileUpload(e, 'residence')); 
        });

    </script>
</body>
</html>
