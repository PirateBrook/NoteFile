
## 有用的Coding记录 ##

- 生成唯一的id
```java
private final String mId = UUID.randomUUID().toString();
```

- Pattern
```java
    // This pattern matches the part identifier for MMS part files.
    protected static final Pattern MMS_PART_FILENAME_PATTERN = Pattern.compile("(PART_\\d+_)");
```

- public downloads uri
```java
private static final Uri PUBLICLY_ACCESSIBLE_DOWNlOADS_URI = 
            Uri.parse("content://downloads/public_downloads");
```

- FileUtil.java Copy the data from the source stream to the file.
```java
    /**
     * Copy the data from the source stream to the file.
     *
     * @param inputStream source stream.
     * @param destFile file to which data will be copied.
     * @return true if succeeds; false otherwise.
     */
    public static boolean copyToFile(InputStream inputStream, File destFile) {
        try {
            FileOutputStream fileOutputStream = new FileOutputStream(destFile);
            try {
                byte[] buffer = new byte[4096];
                int bytesRead;
                while ((bytesRead = inputStream.read(buffer)) >= 0) {
                    fileOutputStream.write(buffer, 0, bytesRead);
                }
            } finally {
                fileOutputStream.flush();
                try {
                    fileOutputStream.getFD().sync();
                } catch (IOException e){

                }
                fileOutputStream.close();
            }
            return true;
        } catch (IOException e) {
            return false;
        }
    }
}
```

### get and copy byte array ###
```java
public byte[] getData() {
        if (mData != null) {
            byte[] data = new byte[mData.length];
            System.arraycopy(mData, 0, data, 0, mData.length);
        }
        return mData;
    }
```

### Uri为空时，主动抛出异常 ###
```java
if (uri == null) {
            throw new IllegalArgumentException("uri may not be null.");
        }
```


