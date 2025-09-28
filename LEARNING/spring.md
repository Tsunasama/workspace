for file in 【动漫极】Stray\ Dogs\ \#*.mp4; do
    # 提取集数
    episode=$(echo "$file" | grep -oP '#\K\d+')
    # 构建新文件名
    new_name="${episode}.mp4"
    # 重命名文件
    mv "$file" "$new_name"
    echo "已将 $file 重命名为 $new_name"
done
