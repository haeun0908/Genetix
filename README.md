# Genetix 메모
## 1. Generation(학습횟수) UI 추가 과정
### 1. UI Text 추가
- Hierarchy 창에서 UI > Legacy > Text를 추가 (간단한 UI 텍스트만 표시하기 위해 Legacy UI Text를 사용)<br>
  Text 이름은 GenerationText로 설정<br>
![image](https://github.com/user-attachments/assets/a49d4ef6-a186-4a23-8e5d-d5671cb150ce)

- RectTransform에서 위치를 조정해서 화면에 잘 보이게 배치<br>
![image](https://github.com/user-attachments/assets/7d30e31e-b104-46e9-9ba4-5748252a5c35)

- Inspector에서 Font Size, Color, Alignment 등을 설정<br>
  (참고: Text는 따로 설정하지 않아도 코드에서 UI의 내용을 업데이트 ➡️ UpdateUI() 메서드)<br>
![image](https://github.com/user-attachments/assets/75e2ab37-4fd4-4d2d-bdbc-ea8c33ad87af)

### 2. GeneticAlgorithm.cs 수정 (UI 연결)
- Text를 업데이트하기 위해 GeneticAlgorithm.cs에서 Text 변수를 선언하고 업데이트 로직을 추가 (추가된 부분은 주석 처리하며, 앞부분에 '추가'를 명시)
```
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.UI; // 추가: Legacy UI 사용을 위한 네임스페이스 추가

public class GeneticAlgorithm : MonoBehaviour
{
    public GameObject carPrefab;
    public int populationSize = 16;  // 예시로 16대 차량
    private List<GameObject> cars = new List<GameObject>();
    private int generation = 0;

    public CameraMove cameraMove;
    public Text generationText; // 추가: UI 요소 (학습 횟수를 표시할 Legacy Text)

    private float waitTime = 2f;
    private float elapsedTime = 0f;
    private bool waiting = true;

    void Start()
    {
        if (cameraMove == null)
        {
            cameraMove = Camera.main.GetComponent<CameraMove>();
        }

        if (cameraMove == null)
        {
            Debug.LogError("CameraMove component is not assigned or found on the main camera!");
        }

        if (generationText == null)
        {
            Debug.LogError("GenerationText UI가 할당되지 않았습니다!"); // 추가: UI Text가 연결되지 않았을 경우 오류 메시지 출력
        }

        Debug.Log("Initializing population...");
        InitializePopulation();
        UpdateUI(); // 추가: UI 업데이트
    }

    void Update()
    {
        if (waiting)
        {
            elapsedTime += Time.deltaTime;
            if (elapsedTime >= waitTime)
            {
                waiting = false;
                Debug.Log("지연 시간 종료! 차량 움직임 체크 시작");
            }
            return;
        }

        if (AllCarsStopped())
        {
            Debug.Log("All cars stopped. Proceeding to create next generation...");
            GameObject bestCar = FindBestCar();
            if (bestCar != null)
            {
                Debug.Log($"Best car score: {bestCar.GetComponent<CarMove>().GetScore()}");
            }

            CreateNextGeneration(bestCar);
        }
    }

    bool AllCarsStopped()
    {
        bool allStopped = true;
        for (int i = 0; i < cars.Count; i++)
        {
            GameObject car = cars[i];
            CarMove carMove = car.GetComponent<CarMove>();

            if (carMove != null)
            {
                Debug.Log($"Car {i} stopped: {carMove.isStopped}");
                if (!carMove.isStopped)
                {
                    allStopped = false;
                }
            }
        }
        return allStopped;
    }

    GameObject FindBestCar()
    {
        GameObject bestCar = null;
        int highestScore = -1;

        foreach (GameObject car in cars)
        {
            CarMove carMove = car.GetComponent<CarMove>();
            if (carMove != null && carMove.GetScore() > highestScore)
            {
                highestScore = carMove.GetScore();
                bestCar = car;
            }
        }

        return bestCar;
    }

    void CreateNextGeneration(GameObject bestCar)
    {
        Debug.Log("Creating next generation...");
        generation++;
        UpdateUI(); // 추가: UI 업데이트
        List<GameObject> newCars = new List<GameObject>();

        for (int i = 0; i < populationSize; i++)
        {
            Vector3 spawnPos = new Vector3(-6f, -6.1f, 0f);
            GameObject newCar = Instantiate(carPrefab, spawnPos, Quaternion.identity);

            // 유전자 돌연변이 적용 (첫 세대부터 다르게 설정)
            CarMove bestMove = bestCar != null ? bestCar.GetComponent<CarMove>() : null;
            CarMove newMove = newCar.GetComponent<CarMove>();

            if (bestMove != null && newMove != null)
            {
                // 첫 세대는 무작위로 설정 (최소값 ~ 최대값 사이에서 랜덤 값 부여)
                newMove.handling = Random.Range(100, 2000); // handling: 100 ~ 2000 사이
                newMove.steeringSensitivity = Random.Range(0.1f, 3f); // steeringSensitivity: 0.1 ~ 3 사이

                // 돌연변이: bestCar의 유전자에 약간의 변화를 주어서 다르게 설정
                if (Random.Range(0f, 1f) < 0.1f)  // 10% 확률로 돌연변이 발생
                {
                    newMove.handling += Random.Range(-50, 51); // +- 50의 변화를 줄 수 있음
                    newMove.steeringSensitivity += Random.Range(-0.2f, 0.2f);
                }
            }
            else
            {
                // 첫 세대에서만 무작위로 설정
                newMove.handling = Random.Range(100, 2000);
                newMove.steeringSensitivity = Random.Range(0.1f, 3f);
            }

            // 범위 제한
            newMove.handling = Mathf.Clamp(newMove.handling, 100, 2000);
            newMove.steeringSensitivity = Mathf.Clamp(newMove.steeringSensitivity, 0.1f, 3f);

            newCars.Add(newCar);
        }

        foreach (GameObject car in cars)
        {
            if (car != null)
            {
                Destroy(car);
            }
        }

        cars.Clear();
        cars = newCars;

        if (cameraMove != null)
        {
            cameraMove.SetCarList(cars.ToArray());
        }

        RestartCars();

        // 지연 시간 초기화
        waiting = true;
        elapsedTime = 0f;
    }

    void RestartCars()
    {
        foreach (GameObject car in cars)
        {
            CarMove carMove = car.GetComponent<CarMove>();
            if (carMove != null)
            {
                carMove.isStopped = false;
            }
        }
    }

    // 추가: UI 업데이트 (현재 세대 수 표시)
    void UpdateUI()
    {
        if (generationText != null)
        {
            generationText.text = $"Generation: {generation}"; // 세대 정보 표시
        }
    }

    void InitializePopulation()
    {
        for (int i = 0; i < populationSize; i++)
        {
            // 모든 차량이 동일한 시작 위치에서 시작
            Vector3 spawnPos = new Vector3(-6f, -6.1f, 0f);
            GameObject car = Instantiate(carPrefab, spawnPos, Quaternion.identity);

            // 첫 세대에서 차량들의 유전자는 무작위로 다르게 설정
            CarMove carMove = car.GetComponent<CarMove>();
            if (carMove != null)
            {
                carMove.handling = Random.Range(100, 2000); // handling: 100 ~ 2000 사이
                carMove.steeringSensitivity = Random.Range(0.1f, 3f); // steeringSensitivity: 0.1 ~ 3 사이
            }

            cars.Add(car);
        }

        if (cameraMove != null)
        {
            cameraMove.SetCarList(cars.ToArray());
            cameraMove.InitializePosition();
        }
        else
        {
            Debug.LogError("CameraMove is not assigned, unable to initialize camera.");
        }
    }
}
```

### 3. UI 연결
- GeneticManager(GeneticAlgorithm 컴포넌트) 선택<br>
![image](https://github.com/user-attachments/assets/da05ed89-baca-48c1-bd31-801270ef40c0)

- GenerationText 변수에 Text 오브젝트를 드래그<br>
![image](https://github.com/user-attachments/assets/843c5475-783d-4d5c-b920-9f381773ca55)
