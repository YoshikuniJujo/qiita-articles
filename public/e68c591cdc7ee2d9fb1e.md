---
title: 巡回冗長検査(CRC-32)のHaskellでのゴルフ的コード
tags:
  - Haskell
  - crc
private: false
updated_at: '2015-12-25T20:09:00+09:00'
id: e68c591cdc7ee2d9fb1e
organization_url_name: null
slide: false
ignorePublish: false
---
(説明はあとで書く)

ゴルフっぽいけどゴルフではないコード。

```hs:CRC.hs
module CRC (check, calc) where

import Control.Applicative ((<$>))
import Control.Arrow (first, (&&&))
import Data.Array (Array, (!), listArray)
import Data.Bits (complement, xor, shiftR, testBit)
import Data.Bool (bool)
import Data.Word (Word8, Word32)
import qualified Data.ByteString as BS (ByteString, foldl')

check :: BS.ByteString -> Bool
check = (== 0x2144df1c) . calc

calc :: BS.ByteString -> Word32
calc = complement . BS.foldl' (flip st) 0xffffffff
        where st b = uncurry xor . first ((table !) . (xor b))
                . (fromIntegral &&& (`shiftR` 8))

table :: Array Word8 Word32
table = listArray (0, 255) . (<$> [0 .. 255]) . ((!! 8) .) . iterate $
        uncurry (bool id $ xor 0xedb88320) . ((`testBit` 0) &&& (`shiftR` 1))
```
