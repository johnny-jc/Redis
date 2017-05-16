# Redis
redis

<?php

namespace common\helpers;

use Yii;
use yii\redis\Connection;

/**
 * Redis 操作类
 * @package common\helpers
 */
class RedisHelper
{

    /**
     * 设置一个redis hash 类型的缓存
     *
     * @param string $key hash缓存key
     * @param array $hash
     *--[
     *     1 => ['username' => 'xxx', 'id' => 2],
     *     2 => ['username' => '小红', 'id' => 3]
     * ]
     * @return array|bool|null|string
     */
    public static function setHash($key, array $hash)
    {
        $key = static::_buildKey($key);
        $params = [$key];
        foreach ($hash as $hashKey => $hashValue) {
            $params[] = $hashKey;
            $params[] = is_array($hashValue) ? json_encode($hashValue) : $hashValue;
        }

        return static::_redis()->executeCommand('HMSET', $params);
    }

    /**
     * 获得一个或多个hash数据
     *
     * @param string $key  hash缓存key
     * @param array $fields
     * --不管是取一个hash还是多个hash都必须传递数组
     * --如果取到数据，不管一条还是多条始终返回数组
     * @return array|bool|null|string
     */
    public static function getValueByHash($key, array $fields)
    {
        $key = static::_buildKey($key);

        $params = [$key];
        if (!is_array($fields)) {
            return null;
        }
        $params = array_merge($params, $fields);
        $result = static::_redis()->executeCommand('HMGET', $params);

        if ($result[0] === null) {
            return null;
        }
        foreach ($result as &$item) {
            $item = json_decode($item, true);
        }

        return $result;
    }

    /**
     * 获得所有hash中所有数据
     * @param $key
     * @return array|bool|null|string
     */
    public static function getHashAll($key)
    {
        $key = static::_buildKey($key);
        $params = [$key];
        $cacheData = static::_redis()->executeCommand('HGETALL', $params);

        if (!empty($cacheData)) {
            $datum = [];
            $index = '';
            foreach ($cacheData as $key => $jsonStr) {
                if ($key % 2 == 0) {
                    $index = $jsonStr;
                    continue;
                }
                $datum[$index] = json_decode($jsonStr, true);
            }
            return $datum;
        }
        return $cacheData;
    }

    /**
     * 移除hash中一个或多个字段的数据
     * @param string $key  hash key
     * @param array $fields 要移除的字段
     * @return array|bool|int|null|string 成功返回被移除的数量
     */
    public static function removeHash($key, array $fields)
    {
        if (empty($fields)) {
            return 0;
        }
        $key = static::_buildKey($key);
        $params[] = $key;
        if (is_array($fields)) {
            $params = array_merge($params, $fields);
        } else {
            $params[] = $fields;
        }
        return static::_redis()->executeCommand('HDEL', $params);
    }

    /**
     * 删除一个key
     * @param string $key 缓存key
     * @return bool
     */
    public static function deleteKey($key)
    {
        $key = static::_buildKey($key);
        return (bool) static::_redis()->executeCommand('DEL', [$key]);
    }

    /**
     * 获得redis操作对象
     * @return Connection
     */
    private static function _redis()
    {
        return Yii::$app->redis;
    }

    /**
     * 构建缓存key
     * @param string $key  缓存key
     * @return string
     */
    private static function _buildKey($key)
    {
        return Yii::$app->cache->keyPrefix . $key;
    }
}
