import cv2
import numpy as np

# 計算 LBP 特徵
def lbp_code(image):
    lbp = np.zeros_like(image, dtype=np.uint8)
    for i in range(1, image.shape[0] - 1):
        for j in range(1, image.shape[1] - 1):
            center = image[i, j]
            binary_str = ''.join(['1' if image[i + dx, j + dy] >= center else '0' 
                                  for dx, dy in [(-1, -1), (-1, 0), (-1, 1), (0, 1), (1, 1), (1, 0), (1, -1), (0, -1)]])
            lbp[i, j] = int(binary_str, 2)
    return lbp

# 計算直方圖
def histogram(image):
    hist = cv2.calcHist([image], [0], None, [256], [0, 256])
    hist = cv2.normalize(hist, hist).flatten()
    return hist

# 計算特徵距離
def distance_measure(histA, histB):
    return np.sqrt(np.sum((histA - histB) ** 2))

# 道路提取流程
def extract_road(image):
    # 1. Sobel 邊緣檢測
    sobel_x = cv2.Sobel(image, cv2.CV_64F, 1, 0, ksize=3)
    sobel_y = cv2.Sobel(image, cv2.CV_64F, 0, 1, ksize=3)
    sobel_combined = cv2.magnitude(sobel_x, sobel_y)
    sobel_combined = np.uint8(cv2.normalize(sobel_combined, None, 0, 255, cv2.NORM_MINMAX))
    
    # 2. LBP 編碼
    lbp_image = lbp_code(sobel_combined)
    
    return sobel_combined, lbp_image

# 填滿道路區域
def labeling(original_image, road_mask, lbp_image):
    # 找出輪廓
    contours, _ = cv2.findContours(road_mask, cv2.RETR_EXTERNAL, cv2.CHAIN_APPROX_SIMPLE)
    result_image = cv2.cvtColor(original_image, cv2.COLOR_GRAY2BGR)
    
    # 計算原始影像的 LBP 直方圖
    hist_orig = histogram(lbp_image)
    
    for contour in contours:
        area = cv2.contourArea(contour)
        if area > 1000:  # 面積過小的輪廓不處理
            mask = np.zeros_like(road_mask)
            cv2.drawContours(mask, [contour], -1, 255, thickness=cv2.FILLED)
            
            # 計算掩膜區域的 LBP 直方圖
            masked_image = cv2.bitwise_and(lbp_image, lbp_image, mask=mask)
            hist_masked = histogram(masked_image)
            
            # 計算 LBP 直方圖的歐氏距離
            hist_distance = distance_measure(hist_orig, hist_masked)
            
            # 根據閥值進行填色
            if hist_distance < 0.15:  # 調整此閾值以適應您的影像
                result_image[mask == 255] = (0, 255, 0)
    
    return result_image

#主程式
def main():
    # 讀取單張影像
    image = cv2.imread('road.jpg', cv2.IMREAD_GRAYSCALE)
    
    # 道路提取
    sobel_combined, lbp_image = extract_road(image)
    
    # 自適應二值化
    binary_image = cv2.adaptiveThreshold(sobel_combined, 255, cv2.ADAPTIVE_THRESH_GAUSSIAN_C, 
                                         cv2.THRESH_BINARY, blockSize=15, C=5)
    
    # 形態學操作
    kernel = cv2.getStructuringElement(cv2.MORPH_RECT, (3, 3))
    binary_image = cv2.morphologyEx(binary_image, cv2.MORPH_CLOSE, kernel)
    binary_image = cv2.morphologyEx(binary_image, cv2.MORPH_OPEN, kernel)
    
    # 填滿道路區域
    filled_road_image = labeling(image, binary_image, lbp_image)
    
    # 使用 OpenCV 顯示影像
    cv2.imshow('Sobel Combined', sobel_combined)
    cv2.imshow('LBP Image', lbp_image)
    cv2.imshow('Extracted Road Mask', binary_image)
    cv2.imshow('Filled Road Area', filled_road_image)
    
    # 等待按鍵關閉所有視窗
    cv2.waitKey(0)
    cv2.destroyAllWindows()

if __name__ == "__main__":
    main()