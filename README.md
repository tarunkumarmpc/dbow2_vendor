# dbow2_vendor

`dbow2_vendor` is a ROS 2 vendor package that ships and builds the upstream **DBoW2** library inside your workspace, so downstream SLAM packages can use DBoW2 with:

```cmake
find_package(dbow2_vendor REQUIRED)
```

without manually installing DBoW2 system-wide.

---

## Upstream project

This package vendors the original DBoW2 library by Dorian Gálvez-López.

- Original repository: https://github.com/dorian3d/DBoW2
- Paper: *Bags of Binary Words for Fast Place Recognition in Image Sequences* (IEEE TRO 2012)

The vendored source lives at:

- `vendor/DBoW2`

---

## What this package does

During `colcon build`, `dbow2_vendor`:

1. Builds vendored DBoW2 from `vendor/DBoW2` using `ExternalProject_Add`.
2. Installs DBoW2 headers and libraries into this package install prefix.
3. Exports include dirs, libraries, and OpenCV dependency for downstream packages.
4. Provides an imported target for consumers:
   - `dbow2_vendor::DBoW2`

This gives one stable integration point for all SLAM packages in the workspace.

---

## Why vendor mode (recommended)

Using this vendor package avoids common multi-package issues:

- Multiple DBoW2 copies built with different flags
- ABI mismatch across SLAM packages
- Accidental linking against system `/usr` installs

With `dbow2_vendor`, all consumers resolve to the same DBoW2 build.

---

## Build

From workspace root:

```bash
colcon build --packages-select dbow2_vendor
```

Then source:

```bash
source install/setup.bash
```

---

## Install artifacts

After build/install, key artifacts are available under this package install prefix:

- `include/DBoW2/*` (headers)
- `lib/libDBoW2.so` (or platform equivalent)
- `share/dbow2_vendor/cmake/*` (consumer CMake config)

---

## Downstream package usage

### 1) CMakeLists.txt (consumer)

```cmake
find_package(ament_cmake REQUIRED)
find_package(OpenCV REQUIRED)
find_package(dbow2_vendor REQUIRED)

add_executable(my_slam_node src/my_slam_node.cpp)

target_link_libraries(my_slam_node
  dbow2_vendor::DBoW2
)

ament_target_dependencies(my_slam_node OpenCV)
```

### 2) package.xml (consumer)

```xml
<depend>dbow2_vendor</depend>
<depend>opencv</depend>
```

---

## Minimal C++ example (SLAM-style place recognition)

```cpp
#include <DBoW2/DBoW2.h>
#include <opencv2/core.hpp>
#include <opencv2/features2d.hpp>
#include <iostream>
#include <vector>

int main()
{
  // 1) Load vocabulary (trained offline)
  OrbVocabulary voc;
  if (!voc.load("orb_vocab.yml.gz")) {
    std::cerr << "Failed to load vocabulary" << std::endl;
    return 1;
  }

  // 2) Create database for loop/place queries
  OrbDatabase db(voc, false, 0);

  // 3) Example: extract ORB descriptors from an image
  cv::Mat image = cv::Mat::zeros(480, 640, CV_8UC1);
  auto orb = cv::ORB::create(1200);

  std::vector<cv::KeyPoint> keypoints;
  cv::Mat descriptors; // Nx32 CV_8U
  orb->detectAndCompute(image, cv::noArray(), keypoints, descriptors);

  // 4) Convert descriptor matrix rows to vector<cv::Mat>
  std::vector<cv::Mat> features;
  features.reserve(static_cast<size_t>(descriptors.rows));
  for (int r = 0; r < descriptors.rows; ++r) {
    features.emplace_back(descriptors.row(r));
  }

  // 5) Add frame to DB and query
  db.add(features);

  DBoW2::QueryResults results;
  db.query(features, results, 4); // top-4 matches

  std::cout << "Matches: " << results.size() << std::endl;
  for (const auto &res : results) {
    std::cout << "entry_id=" << res.Id << " score=" << res.Score << std::endl;
  }

  return 0;
}
```

---

## Notes for SLAM integration

- Train and version your vocabulary file outside runtime.
- Keep descriptor type consistent (`ORB` descriptors with `OrbVocabulary`/`OrbDatabase`).
- In multi-node systems, centralize vocabulary loading and reuse where possible.

---

## Maintenance tips

- Keep only one DBoW2 provider in the workspace (`dbow2_vendor`).
- If an old standalone DBoW2 package exists, keep it ignored for colcon builds.
- When updating upstream DBoW2, update `vendor/DBoW2` as one atomic change and test all SLAM consumers.

---

## License

- `dbow2_vendor`: package wrapper metadata/build glue.
- Upstream DBoW2 license: see `vendor/DBoW2/LICENSE.txt`.
