version = "1.0.9"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "certifi" },
    { name = "h11" },
]
sdist = { url = "https://files.pythonhosted.org/packages/06/94/82699a10bca87a5556c9c59b5963f2d039dbd239f25bc2a63907a05a14cb/httpcore-1.0.9.tar.gz", hash = "sha256:6e34463af53fd2ab5d807f399a9b45ea31c3dfa2276f15a2c3f00afff6e176e8", size = 85484, upload-time = "2025-04-24T22:06:22.219Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/7e/f5/f66802a942d491edb555dd61e3a9961140fd64c90bce1eafd741609d334d/httpcore-1.0.9-py3-none-any.whl", hash = "sha256:2d400746a40668fc9dec9810239072b40b4484b640a8c38fd654a024c7a1bf55", size = 78784, upload-time = "2025-04-24T22:06:20.566Z" },
]

[[package]]
name = "httpx"
version = "0.28.1"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "anyio" },
    { name = "certifi" },
    { name = "httpcore" },
    { name = "idna" },
]
sdist = { url = "https://files.pythonhosted.org/packages/b1/df/48c586a5fe32a0f01324ee087459e112ebb7224f646c0b5023f5e79e9956/httpx-0.28.1.tar.gz", hash = "sha256:75e98c5f16b0f35b567856f597f06ff2270a374470a5c2392242528e3e3e42fc", size = 141406, upload-time = "2024-12-06T15:37:23.222Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/2a/39/e50c7c3a983047577ee07d2a9e53faf5a69493943ec3f6a384bdc792deb2/httpx-0.28.1-py3-none-any.whl", hash = "sha256:d909fcccc110f8c7faf814ca82a9a4d816bc5a6dbfea25d6591d6985b8ba59ad", size = 73517, upload-time = "2024-12-06T15:37:21.509Z" },
]

[[package]]
name = "idna"
version = "3.10"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/f1/70/7703c29685631f5a7590aa73f1f1d3fa9a380e654b86af429e0934a32f7d/idna-3.10.tar.gz", hash = "sha256:12f65c9b470abda6dc35cf8e63cc574b1c52b11df2c86030af0ac09b01b13ea9", size = 190490, upload-time = "2024-09-15T18:07:39.745Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/76/c6/c88e154df9c4e1a2a66ccf0005a88dfb2650c1dffb6f5ce603dfbd452ce3/idna-3.10-py3-none-any.whl", hash = "sha256:946d195a0d259cbba61165e88e65941f16e9b36ea6ddb97f00452bae8b1287d3", size = 70442, upload-time = "2024-09-15T18:07:37.964Z" },
]

[[package]]
name = "iniconfig"
version = "2.1.0"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/f2/97/ebf4da567aa6827c909642694d71c9fcf53e5b504f2d96afea02718862f3/iniconfig-2.1.0.tar.gz", hash = "sha256:3abbd2e30b36733fee78f9c7f7308f2d0050e88f0087fd25c2645f63c773e1c7", size = 4793, upload-time = "2025-03-19T20:09:59.721Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/2c/e1/e6716421ea10d38022b952c159d5161ca1193197fb744506875fbb87ea7b/iniconfig-2.1.0-py3-none-any.whl", hash = "sha256:9deba5723312380e77435581c6bf4935c94cbfab9b1ed33ef8d238ea168eb760", size = 6050, upload-time = "2025-03-19T20:10:01.071Z" },
]

[[package]]
name = "langgraph-sdk"
source = { editable = "." }
dependencies = [
    { name = "httpx" },
    { name = "orjson" },
]

[package.dev-dependencies]
dev = [
    { name = "codespell" },
    { name = "mypy" },
    { name = "pytest" },
    { name = "pytest-asyncio" },
    { name = "pytest-mock" },
    { name = "pytest-watch" },
    { name = "ruff" },
]
lint = [
    { name = "codespell" },
    { name = "mypy" },
    { name = "ruff" },
]
test = [
    { name = "pytest" },
    { name = "pytest-asyncio" },
    { name = "pytest-mock" },
    { name = "pytest-watch" },
]

[package.metadata]
requires-dist = [
    { name = "httpx", specifier = ">=0.25.2" },
    { name = "orjson", specifier = ">=3.10.1" },
]

[package.metadata.requires-dev]
dev = [
    { name = "codespell" },
    { name = "mypy" },
    { name = "pytest" },
    { name = "pytest-asyncio" },
    { name = "pytest-mock" },
    { name = "pytest-watch" },
    { name = "ruff" },
]
lint = [
    { name = "codespell" },
    { name = "mypy" },
    { name = "ruff" },
]
test = [
    { name = "pytest" },
    { name = "pytest-asyncio" },
    { name = "pytest-mock" },
    { name = "pytest-watch" },
]

[[package]]
name = "mypy"
version = "1.18.2"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "mypy-extensions" },
    { name = "pathspec" },
    { name = "tomli", marker = "python_full_version < '3.11'" },
    { name = "typing-extensions" },
]
sdist = { url = "https://files.pythonhosted.org/packages/c0/77/8f0d0001ffad290cef2f7f216f96c814866248a0b92a722365ed54648e7e/mypy-1.18.2.tar.gz", hash = "sha256:06a398102a5f203d7477b2923dda3634c36727fa5c237d8f859ef90c42a9924b", size = 3448846, upload-time = "2025-09-19T00:11:10.519Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/03/6f/657961a0743cff32e6c0611b63ff1c1970a0b482ace35b069203bf705187/mypy-1.18.2-cp310-cp310-macosx_10_9_x86_64.whl", hash = "sha256:c1eab0cf6294dafe397c261a75f96dc2c31bffe3b944faa24db5def4e2b0f77c", size = 12807973, upload-time = "2025-09-19T00:10:35.282Z" },
    { url = "https://files.pythonhosted.org/packages/10/e9/420822d4f661f13ca8900f5fa239b40ee3be8b62b32f3357df9a3045a08b/mypy-1.18.2-cp310-cp310-macosx_11_0_arm64.whl", hash = "sha256:7a780ca61fc239e4865968ebc5240bb3bf610ef59ac398de9a7421b54e4a207e", size = 11896527, upload-time = "2025-09-19T00:10:55.791Z" },
    { url = "https://files.pythonhosted.org/packages/aa/73/a05b2bbaa7005f4642fcfe40fb73f2b4fb6bb44229bd585b5878e9a87ef8/mypy-1.18.2-cp310-cp310-manylinux2014_aarch64.manylinux_2_17_aarch64.manylinux_2_28_aarch64.whl", hash = "sha256:448acd386266989ef11662ce3c8011fd2a7b632e0ec7d61a98edd8e27472225b", size = 12507004, upload-time = "2025-09-19T00:11:05.411Z" },
    { url = "https://files.pythonhosted.org/packages/4f/01/f6e4b9f0d031c11ccbd6f17da26564f3a0f3c4155af344006434b0a05a9d/mypy-1.18.2-cp310-cp310-manylinux2014_x86_64.manylinux_2_17_x86_64.manylinux_2_28_x86_64.whl", hash = "sha256:f9e171c465ad3901dc652643ee4bffa8e9fef4d7d0eece23b428908c77a76a66", size = 13245947, upload-time = "2025-09-19T00:10:46.923Z" },
    { url = "https://files.pythonhosted.org/packages/d7/97/19727e7499bfa1ae0773d06afd30ac66a58ed7437d940c70548634b24185/mypy-1.18.2-cp310-cp310-musllinux_1_2_x86_64.whl", hash = "sha256:592ec214750bc00741af1f80cbf96b5013d81486b7bb24cb052382c19e40b428", size = 13499217, upload-time = "2025-09-19T00:09:39.472Z" },
    { url = "https://files.pythonhosted.org/packages/9f/4f/90dc8c15c1441bf31cf0f9918bb077e452618708199e530f4cbd5cede6ff/mypy-1.18.2-cp310-cp310-win_amd64.whl", hash = "sha256:7fb95f97199ea11769ebe3638c29b550b5221e997c63b14ef93d2e971606ebed", size = 9766753, upload-time = "2025-09-19T00:10:49.161Z" },
    { url = "https://files.pythonhosted.org/packages/88/87/cafd3ae563f88f94eec33f35ff722d043e09832ea8530ef149ec1efbaf08/mypy-1.18.2-cp311-cp311-macosx_10_9_x86_64.whl", hash = "sha256:807d9315ab9d464125aa9fcf6d84fde6e1dc67da0b6f80e7405506b8ac72bc7f", size = 12731198, upload-time = "2025-09-19T00:09:44.857Z" },
    { url = "https://files.pythonhosted.org/packages/0f/e0/1e96c3d4266a06d4b0197ace5356d67d937d8358e2ee3ffac71faa843724/mypy-1.18.2-cp311-cp311-macosx_11_0_arm64.whl", hash = "sha256:776bb00de1778caf4db739c6e83919c1d85a448f71979b6a0edd774ea8399341", size = 11817879, upload-time = "2025-09-19T00:09:47.131Z" },
    { url = "https://files.pythonhosted.org/packages/72/ef/0c9ba89eb03453e76bdac5a78b08260a848c7bfc5d6603634774d9cd9525/mypy-1.18.2-cp311-cp311-manylinux2014_aarch64.manylinux_2_17_aarch64.manylinux_2_28_aarch64.whl", hash = "sha256:1379451880512ffce14505493bd9fe469e0697543717298242574882cf8cdb8d", size = 12427292, upload-time = "2025-09-19T00:10:22.472Z" },
    { url = "https://files.pythonhosted.org/packages/1a/52/ec4a061dd599eb8179d5411d99775bec2a20542505988f40fc2fee781068/mypy-1.18.2-cp311-cp311-manylinux2014_x86_64.manylinux_2_17_x86_64.manylinux_2_28_x86_64.whl", hash = "sha256:1331eb7fd110d60c24999893320967594ff84c38ac6d19e0a76c5fd809a84c86", size = 13163750, upload-time = "2025-09-19T00:09:51.472Z" },
    { url = "https://files.pythonhosted.org/packages/c4/5f/2cf2ceb3b36372d51568f2208c021870fe7834cf3186b653ac6446511839/mypy-1.18.2-cp311-cp311-musllinux_1_2_x86_64.whl", hash = "sha256:3ca30b50a51e7ba93b00422e486cbb124f1c56a535e20eff7b2d6ab72b3b2e37", size = 13351827, upload-time = "2025-09-19T00:09:58.311Z" },
    { url = "https://files.pythonhosted.org/packages/c8/7d/2697b930179e7277529eaaec1513f8de622818696857f689e4a5432e5e27/mypy-1.18.2-cp311-cp311-win_amd64.whl", hash = "sha256:664dc726e67fa54e14536f6e1224bcfce1d9e5ac02426d2326e2bb4e081d1ce8", size = 9757983, upload-time = "2025-09-19T00:10:09.071Z" },
    { url = "https://files.pythonhosted.org/packages/07/06/dfdd2bc60c66611dd8335f463818514733bc763e4760dee289dcc33df709/mypy-1.18.2-cp312-cp312-macosx_10_13_x86_64.whl", hash = "sha256:33eca32dd124b29400c31d7cf784e795b050ace0e1f91b8dc035672725617e34", size = 12908273, upload-time = "2025-09-19T00:10:58.321Z" },
    { url = "https://files.pythonhosted.org/packages/81/14/6a9de6d13a122d5608e1a04130724caf9170333ac5a924e10f670687d3eb/mypy-1.18.2-cp312-cp312-macosx_11_0_arm64.whl", hash = "sha256:a3c47adf30d65e89b2dcd2fa32f3aeb5e94ca970d2c15fcb25e297871c8e4764", size = 11920910, upload-time = "2025-09-19T00:10:20.043Z" },
    { url = "https://files.pythonhosted.org/packages/5f/a9/b29de53e42f18e8cc547e38daa9dfa132ffdc64f7250e353f5c8cdd44bee/mypy-1.18.2-cp312-cp312-manylinux2014_aarch64.manylinux_2_17_aarch64.manylinux_2_28_aarch64.whl", hash = "sha256:5d6c838e831a062f5f29d11c9057c6009f60cb294fea33a98422688181fe2893", size = 12465585, upload-time = "2025-09-19T00:10:33.005Z" },
    { url = "https://files.pythonhosted.org/packages/77/ae/6c3d2c7c61ff21f2bee938c917616c92ebf852f015fb55917fd6e2811db2/mypy-1.18.2-cp312-cp312-manylinux2014_x86_64.manylinux_2_17_x86_64.manylinux_2_28_x86_64.whl", hash = "sha256:01199871b6110a2ce984bde85acd481232d17413868c9807e95c1b0739a58914", size = 13348562, upload-time = "2025-09-19T00:10:11.51Z" },
    { url = "https://files.pythonhosted.org/packages/4d/31/aec68ab3b4aebdf8f36d191b0685d99faa899ab990753ca0fee60fb99511/mypy-1.18.2-cp312-cp312-musllinux_1_2_x86_64.whl", hash = "sha256:a2afc0fa0b0e91b4599ddfe0f91e2c26c2b5a5ab263737e998d6817874c5f7c8", size = 13533296, upload-time = "2025-09-19T00:10:06.568Z" },
    { url = "https://files.pythonhosted.org/packages/9f/83/abcb3ad9478fca3ebeb6a5358bb0b22c95ea42b43b7789c7fb1297ca44f4/mypy-1.18.2-cp312-cp312-win_amd64.whl", hash = "sha256:d8068d0afe682c7c4897c0f7ce84ea77f6de953262b12d07038f4d296d547074", size = 9828828, upload-time = "2025-09-19T00:10:28.203Z" },
    { url = "https://files.pythonhosted.org/packages/5f/04/7f462e6fbba87a72bc8097b93f6842499c428a6ff0c81dd46948d175afe8/mypy-1.18.2-cp313-cp313-macosx_10_13_x86_64.whl", hash = "sha256:07b8b0f580ca6d289e69209ec9d3911b4a26e5abfde32228a288eb79df129fcc", size = 12898728, upload-time = "2025-09-19T00:10:01.33Z" },
    { url = "https://files.pythonhosted.org/packages/99/5b/61ed4efb64f1871b41fd0b82d29a64640f3516078f6c7905b68ab1ad8b13/mypy-1.18.2-cp313-cp313-macosx_11_0_arm64.whl", hash = "sha256:ed4482847168439651d3feee5833ccedbf6657e964572706a2adb1f7fa4dfe2e", size = 11910758, upload-time = "2025-09-19T00:10:42.607Z" },
    { url = "https://files.pythonhosted.org/packages/3c/46/d297d4b683cc89a6e4108c4250a6a6b717f5fa96e1a30a7944a6da44da35/mypy-1.18.2-cp313-cp313-manylinux2014_aarch64.manylinux_2_17_aarch64.manylinux_2_28_aarch64.whl", hash = "sha256:c3ad2afadd1e9fea5cf99a45a822346971ede8685cc581ed9cd4d42eaf940986", size = 12475342, upload-time = "2025-09-19T00:11:00.371Z" },
    { url = "https://files.pythonhosted.org/packages/83/45/4798f4d00df13eae3bfdf726c9244bcb495ab5bd588c0eed93a2f2dd67f3/mypy-1.18.2-cp313-cp313-manylinux2014_x86_64.manylinux_2_17_x86_64.manylinux_2_28_x86_64.whl", hash = "sha256:a431a6f1ef14cf8c144c6b14793a23ec4eae3db28277c358136e79d7d062f62d", size = 13338709, upload-time = "2025-09-19T00:11:03.358Z" },
    { url = "https://files.pythonhosted.org/packages/d7/09/479f7358d9625172521a87a9271ddd2441e1dab16a09708f056e97007207/mypy-1.18.2-cp313-cp313-musllinux_1_2_x86_64.whl", hash = "sha256:7ab28cc197f1dd77a67e1c6f35cd1f8e8b73ed2217e4fc005f9e6a504e46e7ba", size = 13529806, upload-time = "2025-09-19T00:10:26.073Z" },
    { url = "https://files.pythonhosted.org/packages/71/cf/ac0f2c7e9d0ea3c75cd99dff7aec1c9df4a1376537cb90e4c882267ee7e9/mypy-1.18.2-cp313-cp313-win_amd64.whl", hash = "sha256:0e2785a84b34a72ba55fb5daf079a1003a34c05b22238da94fcae2bbe46f3544", size = 9833262, upload-time = "2025-09-19T00:10:40.035Z" },
    { url = "https://files.pythonhosted.org/packages/5a/0c/7d5300883da16f0063ae53996358758b2a2df2a09c72a5061fa79a1f5006/mypy-1.18.2-cp314-cp314-macosx_10_13_x86_64.whl", hash = "sha256:62f0e1e988ad41c2a110edde6c398383a889d95b36b3e60bcf155f5164c4fdce", size = 12893775, upload-time = "2025-09-19T00:10:03.814Z" },
    { url = "https://files.pythonhosted.org/packages/50/df/2cffbf25737bdb236f60c973edf62e3e7b4ee1c25b6878629e88e2cde967/mypy-1.18.2-cp314-cp314-macosx_11_0_arm64.whl", hash = "sha256:8795a039bab805ff0c1dfdb8cd3344642c2b99b8e439d057aba30850b8d3423d", size = 11936852, upload-time = "2025-09-19T00:10:51.631Z" },
    { url = "https://files.pythonhosted.org/packages/be/50/34059de13dd269227fb4a03be1faee6e2a4b04a2051c82ac0a0b5a773c9a/mypy-1.18.2-cp314-cp314-manylinux2014_aarch64.manylinux_2_17_aarch64.manylinux_2_28_aarch64.whl", hash = "sha256:6ca1e64b24a700ab5ce10133f7ccd956a04715463d30498e64ea8715236f9c9c", size = 12480242, upload-time = "2025-09-19T00:11:07.955Z" },
    { url = "https://files.pythonhosted.org/packages/5b/11/040983fad5132d85914c874a2836252bbc57832065548885b5bb5b0d4359/mypy-1.18.2-cp314-cp314-manylinux2014_x86_64.manylinux_2_17_x86_64.manylinux_2_28_x86_64.whl", hash = "sha256:d924eef3795cc89fecf6bedc6ed32b33ac13e8321344f6ddbf8ee89f706c05cb", size = 13326683, upload-time = "2025-09-19T00:09:55.572Z" },
    { url = "https://files.pythonhosted.org/packages/e9/ba/89b2901dd77414dd7a8c8729985832a5735053be15b744c18e4586e506ef/mypy-1.18.2-cp314-cp314-musllinux_1_2_x86_64.whl", hash = "sha256:20c02215a080e3a2be3aa50506c67242df1c151eaba0dcbc1e4e557922a26075", size = 13514749, upload-time = "2025-09-19T00:10:44.827Z" },
    { url = "https://files.pythonhosted.org/packages/25/bc/cc98767cffd6b2928ba680f3e5bc969c4152bf7c2d83f92f5a504b92b0eb/mypy-1.18.2-cp314-cp314-win_amd64.whl", hash = "sha256:749b5f83198f1ca64345603118a6f01a4e99ad4bf9d103ddc5a3200cc4614adf", size = 9982959, upload-time = "2025-09-19T00:10:37.344Z" },
    { url = "https://files.pythonhosted.org/packages/87/e3/be76d87158ebafa0309946c4a73831974d4d6ab4f4ef40c3b53a385a66fd/mypy-1.18.2-py3-none-any.whl", hash = "sha256:22a1748707dd62b58d2ae53562ffc4d7f8bcc727e8ac7cbc69c053ddc874d47e", size = 2352367, upload-time = "2025-09-19T00:10:15.489Z" },
]

[[package]]
name = "mypy-extensions"
version = "1.1.0"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/a2/6e/371856a3fb9d31ca8dac321cda606860fa4548858c0cc45d9d1d4ca2628b/mypy_extensions-1.1.0.tar.gz", hash = "sha256:52e68efc3284861e772bbcd66823fde5ae21fd2fdb51c62a211403730b916558", size = 6343, upload-time = "2025-04-22T14:54:24.164Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/79/7b/2c79738432f5c924bef5071f933bcc9efd0473bac3b4aa584a6f7c1c8df8/mypy_extensions-1.1.0-py3-none-any.whl", hash = "sha256:1be4cccdb0f2482337c4743e60421de3a356cd97508abadd57d47403e94f5505", size = 4963, upload-time = "2025-04-22T14:54:22.983Z" },
]

[[package]]
name = "orjson"
version = "3.11.3"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/be/4d/8df5f83256a809c22c4d6792ce8d43bb503be0fb7a8e4da9025754b09658/orjson-3.11.3.tar.gz", hash = "sha256:1c0603b1d2ffcd43a411d64797a19556ef76958aef1c182f22dc30860152a98a", size = 5482394, upload-time = "2025-08-26T17:46:43.171Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/9b/64/4a3cef001c6cd9c64256348d4c13a7b09b857e3e1cbb5185917df67d8ced/orjson-3.11.3-cp310-cp310-macosx_10_15_x86_64.macosx_11_0_arm64.macosx_10_15_universal2.whl", hash = "sha256:29cb1f1b008d936803e2da3d7cba726fc47232c45df531b29edf0b232dd737e7", size = 238600, upload-time = "2025-08-26T17:44:36.875Z" },
    { url = "https://files.pythonhosted.org/packages/10/ce/0c8c87f54f79d051485903dc46226c4d3220b691a151769156054df4562b/orjson-3.11.3-cp310-cp310-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:97dceed87ed9139884a55db8722428e27bd8452817fbf1869c58b49fecab1120", size = 123526, upload-time = "2025-08-26T17:44:39.574Z" },
    { url = "https://files.pythonhosted.org/packages/ef/d0/249497e861f2d438f45b3ab7b7b361484237414945169aa285608f9f7019/orjson-3.11.3-cp310-cp310-manylinux_2_17_armv7l.manylinux2014_armv7l.whl", hash = "sha256:58533f9e8266cb0ac298e259ed7b4d42ed3fa0b78ce76860626164de49e0d467", size = 128075, upload-time = "2025-08-26T17:44:40.672Z" },
    { url = "https://files.pythonhosted.org/packages/e5/64/00485702f640a0fd56144042a1ea196469f4a3ae93681871564bf74fa996/orjson-3.11.3-cp310-cp310-manylinux_2_17_i686.manylinux2014_i686.whl", hash = "sha256:0c212cfdd90512fe722fa9bd620de4d46cda691415be86b2e02243242ae81873", size = 130483, upload-time = "2025-08-26T17:44:41.788Z" },
    { url = "https://files.pythonhosted.org/packages/64/81/110d68dba3909171bf3f05619ad0cf187b430e64045ae4e0aa7ccfe25b15/orjson-3.11.3-cp310-cp310-manylinux_2_17_ppc64le.manylinux2014_ppc64le.whl", hash = "sha256:5ff835b5d3e67d9207343effb03760c00335f8b5285bfceefd4dc967b0e48f6a", size = 132539, upload-time = "2025-08-26T17:44:43.12Z" },
    { url = "https://files.pythonhosted.org/packages/79/92/dba25c22b0ddfafa1e6516a780a00abac28d49f49e7202eb433a53c3e94e/orjson-3.11.3-cp310-cp310-manylinux_2_17_s390x.manylinux2014_s390x.whl", hash = "sha256:f5aa4682912a450c2db89cbd92d356fef47e115dffba07992555542f344d301b", size = 135390, upload-time = "2025-08-26T17:44:44.199Z" },
    { url = "https://files.pythonhosted.org/packages/44/1d/ca2230fd55edbd87b58a43a19032d63a4b180389a97520cc62c535b726f9/orjson-3.11.3-cp310-cp310-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:d7d18dd34ea2e860553a579df02041845dee0af8985dff7f8661306f95504ddf", size = 132966, upload-time = "2025-08-26T17:44:45.719Z" },
    { url = "https://files.pythonhosted.org/packages/6e/b9/96bbc8ed3e47e52b487d504bd6861798977445fbc410da6e87e302dc632d/orjson-3.11.3-cp310-cp310-musllinux_1_2_aarch64.whl", hash = "sha256:d8b11701bc43be92ea42bd454910437b355dfb63696c06fe953ffb40b5f763b4", size = 131349, upload-time = "2025-08-26T17:44:46.862Z" },
    { url = "https://files.pythonhosted.org/packages/c4/3c/418fbd93d94b0df71cddf96b7fe5894d64a5d890b453ac365120daec30f7/orjson-3.11.3-cp310-cp310-musllinux_1_2_armv7l.whl", hash = "sha256:90368277087d4af32d38bd55f9da2ff466d25325bf6167c8f382d8ee40cb2bbc", size = 404087, upload-time = "2025-08-26T17:44:48.079Z" },
    { url = "https://files.pythonhosted.org/packages/5b/a9/2bfd58817d736c2f63608dec0c34857339d423eeed30099b126562822191/orjson-3.11.3-cp310-cp310-musllinux_1_2_i686.whl", hash = "sha256:fd7ff459fb393358d3a155d25b275c60b07a2c83dcd7ea962b1923f5a1134569", size = 146067, upload-time = "2025-08-26T17:44:49.302Z" },
    { url = "https://files.pythonhosted.org/packages/33/ba/29023771f334096f564e48d82ed855a0ed3320389d6748a9c949e25be734/orjson-3.11.3-cp310-cp310-musllinux_1_2_x86_64.whl", hash = "sha256:f8d902867b699bcd09c176a280b1acdab57f924489033e53d0afe79817da37e6", size = 135506, upload-time = "2025-08-26T17:44:50.558Z" },
    { url = "https://files.pythonhosted.org/packages/39/62/b5a1eca83f54cb3aa11a9645b8a22f08d97dbd13f27f83aae7c6666a0a05/orjson-3.11.3-cp310-cp310-win32.whl", hash = "sha256:bb93562146120bb51e6b154962d3dadc678ed0fce96513fa6bc06599bb6f6edc", size = 136352, upload-time = "2025-08-26T17:44:51.698Z" },
    { url = "https://files.pythonhosted.org/packages/e3/c0/7ebfaa327d9a9ed982adc0d9420dbce9a3fec45b60ab32c6308f731333fa/orjson-3.11.3-cp310-cp310-win_amd64.whl", hash = "sha256:976c6f1975032cc327161c65d4194c549f2589d88b105a5e3499429a54479770", size = 131539, upload-time = "2025-08-26T17:44:52.974Z" },
    { url = "https://files.pythonhosted.org/packages/cd/8b/360674cd817faef32e49276187922a946468579fcaf37afdfb6c07046e92/orjson-3.11.3-cp311-cp311-macosx_10_15_x86_64.macosx_11_0_arm64.macosx_10_15_universal2.whl", hash = "sha256:9d2ae0cc6aeb669633e0124531f342a17d8e97ea999e42f12a5ad4adaa304c5f", size = 238238, upload-time = "2025-08-26T17:44:54.214Z" },
    { url = "https://files.pythonhosted.org/packages/05/3d/5fa9ea4b34c1a13be7d9046ba98d06e6feb1d8853718992954ab59d16625/orjson-3.11.3-cp311-cp311-macosx_15_0_arm64.whl", hash = "sha256:ba21dbb2493e9c653eaffdc38819b004b7b1b246fb77bfc93dc016fe664eac91", size = 127713, upload-time = "2025-08-26T17:44:55.596Z" },
    { url = "https://files.pythonhosted.org/packages/e5/5f/e18367823925e00b1feec867ff5f040055892fc474bf5f7875649ecfa586/orjson-3.11.3-cp311-cp311-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:00f1a271e56d511d1569937c0447d7dce5a99a33ea0dec76673706360a051904", size = 123241, upload-time = "2025-08-26T17:44:57.185Z" },
    { url = "https://files.pythonhosted.org/packages/0f/bd/3c66b91c4564759cf9f473251ac1650e446c7ba92a7c0f9f56ed54f9f0e6/orjson-3.11.3-cp311-cp311-manylinux_2_17_armv7l.manylinux2014_armv7l.whl", hash = "sha256:b67e71e47caa6680d1b6f075a396d04fa6ca8ca09aafb428731da9b3ea32a5a6", size = 127895, upload-time = "2025-08-26T17:44:58.349Z" },
    { url = "https://files.pythonhosted.org/packages/82/b5/dc8dcd609db4766e2967a85f63296c59d4722b39503e5b0bf7fd340d387f/orjson-3.11.3-cp311-cp311-manylinux_2_17_i686.manylinux2014_i686.whl", hash = "sha256:d7d012ebddffcce8c85734a6d9e5f08180cd3857c5f5a3ac70185b43775d043d", size = 130303, upload-time = "2025-08-26T17:44:59.491Z" },
    { url = "https://files.pythonhosted.org/packages/48/c2/d58ec5fd1270b2aa44c862171891adc2e1241bd7dab26c8f46eb97c6c6f1/orjson-3.11.3-cp311-cp311-manylinux_2_17_ppc64le.manylinux2014_ppc64le.whl", hash = "sha256:dd759f75d6b8d1b62012b7f5ef9461d03c804f94d539a5515b454ba3a6588038", size = 132366, upload-time = "2025-08-26T17:45:00.654Z" },
    { url = "https://files.pythonhosted.org/packages/73/87/0ef7e22eb8dd1ef940bfe3b9e441db519e692d62ed1aae365406a16d23d0/orjson-3.11.3-cp311-cp311-manylinux_2_17_s390x.manylinux2014_s390x.whl", hash = "sha256:6890ace0809627b0dff19cfad92d69d0fa3f089d3e359a2a532507bb6ba34efb", size = 135180, upload-time = "2025-08-26T17:45:02.424Z" },
    { url = "https://files.pythonhosted.org/packages/bb/6a/e5bf7b70883f374710ad74faf99bacfc4b5b5a7797c1d5e130350e0e28a3/orjson-3.11.3-cp311-cp311-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:f9d4a5e041ae435b815e568537755773d05dac031fee6a57b4ba70897a44d9d2", size = 132741, upload-time = "2025-08-26T17:45:03.663Z" },
    { url = "https://files.pythonhosted.org/packages/bd/0c/4577fd860b6386ffaa56440e792af01c7882b56d2766f55384b5b0e9d39b/orjson-3.11.3-cp311-cp311-musllinux_1_2_aarch64.whl", hash = "sha256:2d68bf97a771836687107abfca089743885fb664b90138d8761cce61d5625d55", size = 131104, upload-time = "2025-08-26T17:45:04.939Z" },
    { url = "https://files.pythonhosted.org/packages/66/4b/83e92b2d67e86d1c33f2ea9411742a714a26de63641b082bdbf3d8e481af/orjson-3.11.3-cp311-cp311-musllinux_1_2_armv7l.whl", hash = "sha256:bfc27516ec46f4520b18ef645864cee168d2a027dbf32c5537cb1f3e3c22dac1", size = 403887, upload-time = "2025-08-26T17:45:06.228Z" },
    { url = "https://files.pythonhosted.org/packages/6d/e5/9eea6a14e9b5ceb4a271a1fd2e1dec5f2f686755c0fab6673dc6ff3433f4/orjson-3.11.3-cp311-cp311-musllinux_1_2_i686.whl", hash = "sha256:f66b001332a017d7945e177e282a40b6997056394e3ed7ddb41fb1813b83e824", size = 145855, upload-time = "2025-08-26T17:45:08.338Z" },
    { url = "https://files.pythonhosted.org/packages/45/78/8d4f5ad0c80ba9bf8ac4d0fc71f93a7d0dc0844989e645e2074af376c307/orjson-3.11.3-cp311-cp311-musllinux_1_2_x86_64.whl", hash = "sha256:212e67806525d2561efbfe9e799633b17eb668b8964abed6b5319b2f1cfbae1f", size = 135361, upload-time = "2025-08-26T17:45:09.625Z" },
    { url = "https://files.pythonhosted.org/packages/0b/5f/16386970370178d7a9b438517ea3d704efcf163d286422bae3b37b88dbb5/orjson-3.11.3-cp311-cp311-win32.whl", hash = "sha256:6e8e0c3b85575a32f2ffa59de455f85ce002b8bdc0662d6b9c2ed6d80ab5d204", size = 136190, upload-time = "2025-08-26T17:45:10.962Z" },
    { url = "https://files.pythonhosted.org/packages/09/60/db16c6f7a41dd8ac9fb651f66701ff2aeb499ad9ebc15853a26c7c152448/orjson-3.11.3-cp311-cp311-win_amd64.whl", hash = "sha256:6be2f1b5d3dc99a5ce5ce162fc741c22ba9f3443d3dd586e6a1211b7bc87bc7b", size = 131389, upload-time = "2025-08-26T17:45:12.285Z" },
    { url = "https://files.pythonhosted.org/packages/3e/2a/bb811ad336667041dea9b8565c7c9faf2f59b47eb5ab680315eea612ef2e/orjson-3.11.3-cp311-cp311-win_arm64.whl", hash = "sha256:fafb1a99d740523d964b15c8db4eabbfc86ff29f84898262bf6e3e4c9e97e43e", size = 126120, upload-time = "2025-08-26T17:45:13.515Z" },
    { url = "https://files.pythonhosted.org/packages/3d/b0/a7edab2a00cdcb2688e1c943401cb3236323e7bfd2839815c6131a3742f4/orjson-3.11.3-cp312-cp312-macosx_10_15_x86_64.macosx_11_0_arm64.macosx_10_15_universal2.whl", hash = "sha256:8c752089db84333e36d754c4baf19c0e1437012242048439c7e80eb0e6426e3b", size = 238259, upload-time = "2025-08-26T17:45:15.093Z" },
    { url = "https://files.pythonhosted.org/packages/e1/c6/ff4865a9cc398a07a83342713b5932e4dc3cb4bf4bc04e8f83dedfc0d736/orjson-3.11.3-cp312-cp312-macosx_15_0_arm64.whl", hash = "sha256:9b8761b6cf04a856eb544acdd82fc594b978f12ac3602d6374a7edb9d86fd2c2", size = 127633, upload-time = "2025-08-26T17:45:16.417Z" },
    { url = "https://files.pythonhosted.org/packages/6e/e6/e00bea2d9472f44fe8794f523e548ce0ad51eb9693cf538a753a27b8bda4/orjson-3.11.3-cp312-cp312-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:8b13974dc8ac6ba22feaa867fc19135a3e01a134b4f7c9c28162fed4d615008a", size = 123061, upload-time = "2025-08-26T17:45:17.673Z" },
    { url = "https://files.pythonhosted.org/packages/54/31/9fbb78b8e1eb3ac605467cb846e1c08d0588506028b37f4ee21f978a51d4/orjson-3.11.3-cp312-cp312-manylinux_2_17_armv7l.manylinux2014_armv7l.whl", hash = "sha256:f83abab5bacb76d9c821fd5c07728ff224ed0e52d7a71b7b3de822f3df04e15c", size = 127956, upload-time = "2025-08-26T17:45:19.172Z" },
    { url = "https://files.pythonhosted.org/packages/36/88/b0604c22af1eed9f98d709a96302006915cfd724a7ebd27d6dd11c22d80b/orjson-3.11.3-cp312-cp312-manylinux_2_17_i686.manylinux2014_i686.whl", hash = "sha256:e6fbaf48a744b94091a56c62897b27c31ee2da93d826aa5b207131a1e13d4064", size = 130790, upload-time = "2025-08-26T17:45:20.586Z" },
    { url = "https://files.pythonhosted.org/packages/0e/9d/1c1238ae9fffbfed51ba1e507731b3faaf6b846126a47e9649222b0fd06f/orjson-3.11.3-cp312-cp312-manylinux_2_17_ppc64le.manylinux2014_ppc64le.whl", hash = "sha256:bc779b4f4bba2847d0d2940081a7b6f7b5877e05408ffbb74fa1faf4a136c424", size = 132385, upload-time = "2025-08-26T17:45:22.036Z" },
    { url = "https://files.pythonhosted.org/packages/a3/b5/c06f1b090a1c875f337e21dd71943bc9d84087f7cdf8c6e9086902c34e42/orjson-3.11.3-cp312-cp312-manylinux_2_17_s390x.manylinux2014_s390x.whl", hash = "sha256:bd4b909ce4c50faa2192da6bb684d9848d4510b736b0611b6ab4020ea6fd2d23", size = 135305, upload-time = "2025-08-26T17:45:23.4Z" },
    { url = "https://files.pythonhosted.org/packages/a0/26/5f028c7d81ad2ebbf84414ba6d6c9cac03f22f5cd0d01eb40fb2d6a06b07/orjson-3.11.3-cp312-cp312-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:524b765ad888dc5518bbce12c77c2e83dee1ed6b0992c1790cc5fb49bb4b6667", size = 132875, upload-time = "2025-08-26T17:45:25.182Z" },
    { url = "https://files.pythonhosted.org/packages/fe/d4/b8df70d9cfb56e385bf39b4e915298f9ae6c61454c8154a0f5fd7efcd42e/orjson-3.11.3-cp312-cp312-musllinux_1_2_aarch64.whl", hash = "sha256:84fd82870b97ae3cdcea9d8746e592b6d40e1e4d4527835fc520c588d2ded04f", size = 130940, upload-time = "2025-08-26T17:45:27.209Z" },
    { url = "https://files.pythonhosted.org/packages/da/5e/afe6a052ebc1a4741c792dd96e9f65bf3939d2094e8b356503b68d48f9f5/orjson-3.11.3-cp312-cp312-musllinux_1_2_armv7l.whl", hash = "sha256:fbecb9709111be913ae6879b07bafd4b0785b44c1eb5cac8ac76da048b3885a1", size = 403852, upload-time = "2025-08-26T17:45:28.478Z" },
    { url = "https://files.pythonhosted.org/packages/f8/90/7bbabafeb2ce65915e9247f14a56b29c9334003536009ef5b122783fe67e/orjson-3.11.3-cp312-cp312-musllinux_1_2_i686.whl", hash = "sha256:9dba358d55aee552bd868de348f4736ca5a4086d9a62e2bfbbeeb5629fe8b0cc", size = 146293, upload-time = "2025-08-26T17:45:29.86Z" },
    { url = "https://files.pythonhosted.org/packages/27/b3/2d703946447da8b093350570644a663df69448c9d9330e5f1d9cce997f20/orjson-3.11.3-cp312-cp312-musllinux_1_2_x86_64.whl", hash = "sha256:eabcf2e84f1d7105f84580e03012270c7e97ecb1fb1618bda395061b2a84a049", size = 135470, upload-time = "2025-08-26T17:45:31.243Z" },
    { url = "https://files.pythonhosted.org/packages/38/70/b14dcfae7aff0e379b0119c8a812f8396678919c431efccc8e8a0263e4d9/orjson-3.11.3-cp312-cp312-win32.whl", hash = "sha256:3782d2c60b8116772aea8d9b7905221437fdf53e7277282e8d8b07c220f96cca", size = 136248, upload-time = "2025-08-26T17:45:32.567Z" },
    { url = "https://files.pythonhosted.org/packages/35/b8/9e3127d65de7fff243f7f3e53f59a531bf6bb295ebe5db024c2503cc0726/orjson-3.11.3-cp312-cp312-win_amd64.whl", hash = "sha256:79b44319268af2eaa3e315b92298de9a0067ade6e6003ddaef72f8e0bedb94f1", size = 131437, upload-time = "2025-08-26T17:45:34.949Z" },
    { url = "https://files.pythonhosted.org/packages/51/92/a946e737d4d8a7fd84a606aba96220043dcc7d6988b9e7551f7f6d5ba5ad/orjson-3.11.3-cp312-cp312-win_arm64.whl", hash = "sha256:0e92a4e83341ef79d835ca21b8bd13e27c859e4e9e4d7b63defc6e58462a3710", size = 125978, upload-time = "2025-08-26T17:45:36.422Z" },
    { url = "https://files.pythonhosted.org/packages/fc/79/8932b27293ad35919571f77cb3693b5906cf14f206ef17546052a241fdf6/orjson-3.11.3-cp313-cp313-macosx_10_15_x86_64.macosx_11_0_arm64.macosx_10_15_universal2.whl", hash = "sha256:af40c6612fd2a4b00de648aa26d18186cd1322330bd3a3cc52f87c699e995810", size = 238127, upload-time = "2025-08-26T17:45:38.146Z" },
    { url = "https://files.pythonhosted.org/packages/1c/82/cb93cd8cf132cd7643b30b6c5a56a26c4e780c7a145db6f83de977b540ce/orjson-3.11.3-cp313-cp313-macosx_15_0_arm64.whl", hash = "sha256:9f1587f26c235894c09e8b5b7636a38091a9e6e7fe4531937534749c04face43", size = 127494, upload-time = "2025-08-26T17:45:39.57Z" },
    { url = "https://files.pythonhosted.org/packages/a4/b8/2d9eb181a9b6bb71463a78882bcac1027fd29cf62c38a40cc02fc11d3495/orjson-3.11.3-cp313-cp313-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:61dcdad16da5bb486d7227a37a2e789c429397793a6955227cedbd7252eb5a27", size = 123017, upload-time = "2025-08-26T17:45:40.876Z" },
    { url = "https://files.pythonhosted.org/packages/b4/14/a0e971e72d03b509190232356d54c0f34507a05050bd026b8db2bf2c192c/orjson-3.11.3-cp313-cp313-manylinux_2_17_armv7l.manylinux2014_armv7l.whl", hash = "sha256:11c6d71478e2cbea0a709e8a06365fa63da81da6498a53e4c4f065881d21ae8f", size = 127898, upload-time = "2025-08-26T17:45:42.188Z" },
    { url = "https://files.pythonhosted.org/packages/8e/af/dc74536722b03d65e17042cc30ae586161093e5b1f29bccda24765a6ae47/orjson-3.11.3-cp313-cp313-manylinux_2_17_i686.manylinux2014_i686.whl", hash = "sha256:ff94112e0098470b665cb0ed06efb187154b63649403b8d5e9aedeb482b4548c", size = 130742, upload-time = "2025-08-26T17:45:43.511Z" },
    { url = "https://files.pythonhosted.org/packages/62/e6/7a3b63b6677bce089fe939353cda24a7679825c43a24e49f757805fc0d8a/orjson-3.11.3-cp313-cp313-manylinux_2_17_ppc64le.manylinux2014_ppc64le.whl", hash = "sha256:ae8b756575aaa2a855a75192f356bbda11a89169830e1439cfb1a3e1a6dde7be", size = 132377, upload-time = "2025-08-26T17:45:45.525Z" },
    { url = "https://files.pythonhosted.org/packages/fc/cd/ce2ab93e2e7eaf518f0fd15e3068b8c43216c8a44ed82ac2b79ce5cef72d/orjson-3.11.3-cp313-cp313-manylinux_2_17_s390x.manylinux2014_s390x.whl", hash = "sha256:c9416cc19a349c167ef76135b2fe40d03cea93680428efee8771f3e9fb66079d", size = 135313, upload-time = "2025-08-26T17:45:46.821Z" },
    { url = "https://files.pythonhosted.org/packages/d0/b4/f98355eff0bd1a38454209bbc73372ce351ba29933cb3e2eba16c04b9448/orjson-3.11.3-cp313-cp313-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:b822caf5b9752bc6f246eb08124c3d12bf2175b66ab74bac2ef3bbf9221ce1b2", size = 132908, upload-time = "2025-08-26T17:45:48.126Z" },
    { url = "https://files.pythonhosted.org/packages/eb/92/8f5182d7bc2a1bed46ed960b61a39af8389f0ad476120cd99e67182bfb6d/orjson-3.11.3-cp313-cp313-musllinux_1_2_aarch64.whl", hash = "sha256:414f71e3bdd5573893bf5ecdf35c32b213ed20aa15536fe2f588f946c318824f", size = 130905, upload-time = "2025-08-26T17:45:49.414Z" },
    { url = "https://files.pythonhosted.org/packages/1a/60/c41ca753ce9ffe3d0f67b9b4c093bdd6e5fdb1bc53064f992f66bb99954d/orjson-3.11.3-cp313-cp313-musllinux_1_2_armv7l.whl", hash = "sha256:828e3149ad8815dc14468f36ab2a4b819237c155ee1370341b91ea4c8672d2ee", size = 403812, upload-time = "2025-08-26T17:45:51.085Z" },
    { url = "https://files.pythonhosted.org/packages/dd/13/e4a4f16d71ce1868860db59092e78782c67082a8f1dc06a3788aef2b41bc/orjson-3.11.3-cp313-cp313-musllinux_1_2_i686.whl", hash = "sha256:ac9e05f25627ffc714c21f8dfe3a579445a5c392a9c8ae7ba1d0e9fb5333f56e", size = 146277, upload-time = "2025-08-26T17:45:52.851Z" },
    { url = "https://files.pythonhosted.org/packages/8d/8b/bafb7f0afef9344754a3a0597a12442f1b85a048b82108ef2c956f53babd/orjson-3.11.3-cp313-cp313-musllinux_1_2_x86_64.whl", hash = "sha256:e44fbe4000bd321d9f3b648ae46e0196d21577cf66ae684a96ff90b1f7c93633", size = 135418, upload-time = "2025-08-26T17:45:54.806Z" },
    { url = "https://files.pythonhosted.org/packages/60/d4/bae8e4f26afb2c23bea69d2f6d566132584d1c3a5fe89ee8c17b718cab67/orjson-3.11.3-cp313-cp313-win32.whl", hash = "sha256:2039b7847ba3eec1f5886e75e6763a16e18c68a63efc4b029ddf994821e2e66b", size = 136216, upload-time = "2025-08-26T17:45:57.182Z" },
    { url = "https://files.pythonhosted.org/packages/88/76/224985d9f127e121c8cad882cea55f0ebe39f97925de040b75ccd4b33999/orjson-3.11.3-cp313-cp313-win_amd64.whl", hash = "sha256:29be5ac4164aa8bdcba5fa0700a3c9c316b411d8ed9d39ef8a882541bd452fae", size = 131362, upload-time = "2025-08-26T17:45:58.56Z" },
    { url = "https://files.pythonhosted.org/packages/e2/cf/0dce7a0be94bd36d1346be5067ed65ded6adb795fdbe3abd234c8d576d01/orjson-3.11.3-cp313-cp313-win_arm64.whl", hash = "sha256:18bd1435cb1f2857ceb59cfb7de6f92593ef7b831ccd1b9bfb28ca530e539dce", size = 125989, upload-time = "2025-08-26T17:45:59.95Z" },
    { url = "https://files.pythonhosted.org/packages/ef/77/d3b1fef1fc6aaeed4cbf3be2b480114035f4df8fa1a99d2dac1d40d6e924/orjson-3.11.3-cp314-cp314-macosx_10_15_x86_64.macosx_11_0_arm64.macosx_10_15_universal2.whl", hash = "sha256:cf4b81227ec86935568c7edd78352a92e97af8da7bd70bdfdaa0d2e0011a1ab4", size = 238115, upload-time = "2025-08-26T17:46:01.669Z" },
    { url = "https://files.pythonhosted.org/packages/e4/6d/468d21d49bb12f900052edcfbf52c292022d0a323d7828dc6376e6319703/orjson-3.11.3-cp314-cp314-macosx_15_0_arm64.whl", hash = "sha256:bc8bc85b81b6ac9fc4dae393a8c159b817f4c2c9dee5d12b773bddb3b95fc07e", size = 127493, upload-time = "2025-08-26T17:46:03.466Z" },
    { url = "https://files.pythonhosted.org/packages/67/46/1e2588700d354aacdf9e12cc2d98131fb8ac6f31ca65997bef3863edb8ff/orjson-3.11.3-cp314-cp314-manylinux_2_34_aarch64.whl", hash = "sha256:88dcfc514cfd1b0de038443c7b3e6a9797ffb1b3674ef1fd14f701a13397f82d", size = 122998, upload-time = "2025-08-26T17:46:04.803Z" },
    { url = "https://files.pythonhosted.org/packages/3b/94/11137c9b6adb3779f1b34fd98be51608a14b430dbc02c6d41134fbba484c/orjson-3.11.3-cp314-cp314-manylinux_2_34_x86_64.whl", hash = "sha256:d61cd543d69715d5fc0a690c7c6f8dcc307bc23abef9738957981885f5f38229", size = 132915, upload-time = "2025-08-26T17:46:06.237Z" },
    { url = "https://files.pythonhosted.org/packages/10/61/dccedcf9e9bcaac09fdabe9eaee0311ca92115699500efbd31950d878833/orjson-3.11.3-cp314-cp314-musllinux_1_2_aarch64.whl", hash = "sha256:2b7b153ed90ababadbef5c3eb39549f9476890d339cf47af563aea7e07db2451", size = 130907, upload-time = "2025-08-26T17:46:07.581Z" },
    { url = "https://files.pythonhosted.org/packages/0e/fd/0e935539aa7b08b3ca0f817d73034f7eb506792aae5ecc3b7c6e679cdf5f/orjson-3.11.3-cp314-cp314-musllinux_1_2_armv7l.whl", hash = "sha256:7909ae2460f5f494fecbcd10613beafe40381fd0316e35d6acb5f3a05bfda167", size = 403852, upload-time = "2025-08-26T17:46:08.982Z" },
    { url = "https://files.pythonhosted.org/packages/4a/2b/50ae1a5505cd1043379132fdb2adb8a05f37b3e1ebffe94a5073321966fd/orjson-3.11.3-cp314-cp314-musllinux_1_2_i686.whl", hash = "sha256:2030c01cbf77bc67bee7eef1e7e31ecf28649353987775e3583062c752da0077", size = 146309, upload-time = "2025-08-26T17:46:10.576Z" },
    { url = "https://files.pythonhosted.org/packages/cd/1d/a473c158e380ef6f32753b5f39a69028b25ec5be331c2049a2201bde2e19/orjson-3.11.3-cp314-cp314-musllinux_1_2_x86_64.whl", hash = "sha256:a0169ebd1cbd94b26c7a7ad282cf5c2744fce054133f959e02eb5265deae1872", size = 135424, upload-time = "2025-08-26T17:46:12.386Z" },
    { url = "https://files.pythonhosted.org/packages/da/09/17d9d2b60592890ff7382e591aa1d9afb202a266b180c3d4049b1ec70e4a/orjson-3.11.3-cp314-cp314-win32.whl", hash = "sha256:0c6d7328c200c349e3a4c6d8c83e0a5ad029bdc2d417f234152bf34842d0fc8d", size = 136266, upload-time = "2025-08-26T17:46:13.853Z" },
    { url = "https://files.pythonhosted.org/packages/15/58/358f6846410a6b4958b74734727e582ed971e13d335d6c7ce3e47730493e/orjson-3.11.3-cp314-cp314-win_amd64.whl", hash = "sha256:317bbe2c069bbc757b1a2e4105b64aacd3bc78279b66a6b9e51e846e4809f804", size = 131351, upload-time = "2025-08-26T17:46:15.27Z" },
    { url = "https://files.pythonhosted.org/packages/28/01/d6b274a0635be0468d4dbd9cafe80c47105937a0d42434e805e67cd2ed8b/orjson-3.11.3-cp314-cp314-win_arm64.whl", hash = "sha256:e8f6a7a27d7b7bec81bd5924163e9af03d49bbb63013f107b48eb5d16db711bc", size = 125985, upload-time = "2025-08-26T17:46:16.67Z" },
]

[[package]]
name = "packaging"
version = "25.0"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/a1/d4/1fc4078c65507b51b96ca8f8c3ba19e6a61c8253c72794544580a7b6c24d/packaging-25.0.tar.gz", hash = "sha256:d443872c98d677bf60f6a1f2f8c1cb748e8fe762d2bf9d3148b5599295b0fc4f", size = 165727, upload-time = "2025-04-19T11:48:59.673Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/20/12/38679034af332785aac8774540895e234f4d07f7545804097de4b666afd8/packaging-25.0-py3-none-any.whl", hash = "sha256:29572ef2b1f17581046b3a2227d5c611fb25ec70ca1ba8554b24b0e69331a484", size = 66469, upload-time = "2025-04-19T11:48:57.875Z" },
]

[[package]]
name = "pathspec"
version = "0.12.1"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/ca/bc/f35b8446f4531a7cb215605d100cd88b7ac6f44ab3fc94870c120ab3adbf/pathspec-0.12.1.tar.gz", hash = "sha256:a482d51503a1ab33b1c67a6c3813a26953dbdc71c31dacaef9a838c4e29f5712", size = 51043, upload-time = "2023-12-10T22:30:45Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/cc/20/ff623b09d963f88bfde16306a54e12ee5ea43e9b597108672ff3a408aad6/pathspec-0.12.1-py3-none-any.whl", hash = "sha256:a0d503e138a4c123b27490a4f7beda6a01c6f288df0e4a8b79c7eb0dc7b4cc08", size = 31191, upload-time = "2023-12-10T22:30:43.14Z" },
]

[[package]]
name = "pluggy"
version = "1.6.0"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/f9/e2/3e91f31a7d2b083fe6ef3fa267035b518369d9511ffab804f839851d2779/pluggy-1.6.0.tar.gz", hash = "sha256:7dcc130b76258d33b90f61b658791dede3486c3e6bfb003ee5c9bfb396dd22f3", size = 69412, upload-time = "2025-05-15T12:30:07.975Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/54/20/4d324d65cc6d9205fabedc306948156824eb9f0ee1633355a8f7ec5c66bf/pluggy-1.6.0-py3-none-any.whl", hash = "sha256:e920276dd6813095e9377c0bc5566d94c932c33b27a3e3945d8389c374dd4746", size = 20538, upload-time = "2025-05-15T12:30:06.134Z" },
]

[[package]]
name = "pygments"
version = "2.19.2"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/b0/77/a5b8c569bf593b0140bde72ea885a803b82086995367bf2037de0159d924/pygments-2.19.2.tar.gz", hash = "sha256:636cb2477cec7f8952536970bc533bc43743542f70392ae026374600add5b887", size = 4968631, upload-time = "2025-06-21T13:39:12.283Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/c7/21/705964c7812476f378728bdf590ca4b771ec72385c533964653c68e86bdc/pygments-2.19.2-py3-none-any.whl", hash = "sha256:86540386c03d588bb81d44bc3928634ff26449851e99741617ecb9037ee5ec0b", size = 1225217, upload-time = "2025-06-21T13:39:07.939Z" },
]

[[package]]
name = "pytest"
version = "8.4.2"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "colorama", marker = "sys_platform == 'win32'" },
    { name = "exceptiongroup", marker = "python_full_version < '3.11'" },
    { name = "iniconfig" },
    { name = "packaging" },
    { name = "pluggy" },
    { name = "pygments" },
    { name = "tomli", marker = "python_full_version < '3.11'" },
]
sdist = { url = "https://files.pythonhosted.org/packages/a3/5c/00a0e072241553e1a7496d638deababa67c5058571567b92a7eaa258397c/pytest-8.4.2.tar.gz", hash = "sha256:86c0d0b93306b961d58d62a4db4879f27fe25513d4b969df351abdddb3c30e01", size = 1519618, upload-time = "2025-09-04T14:34:22.711Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/a8/a4/20da314d277121d6534b3a980b29035dcd51e6744bd79075a6ce8fa4eb8d/pytest-8.4.2-py3-none-any.whl", hash = "sha256:872f880de3fc3a5bdc88a11b39c9710c3497a547cfa9320bc3c5e62fbf272e79", size = 365750, upload-time = "2025-09-04T14:34:20.226Z" },
]

[[package]]
name = "pytest-asyncio"
version = "1.2.0"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "backports-asyncio-runner", marker = "python_full_version < '3.11'" },
    { name = "pytest" },
    { name = "typing-extensions", marker = "python_full_version < '3.13'" },
]
sdist = { url = "https://files.pythonhosted.org/packages/42/86/9e3c5f48f7b7b638b216e4b9e645f54d199d7abbbab7a64a13b4e12ba10f/pytest_asyncio-1.2.0.tar.gz", hash = "sha256:c609a64a2a8768462d0c99811ddb8bd2583c33fd33cf7f21af1c142e824ffb57", size = 50119, upload-time = "2025-09-12T07:33:53.816Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/04/93/2fa34714b7a4ae72f2f8dad66ba17dd9a2c793220719e736dda28b7aec27/pytest_asyncio-1.2.0-py3-none-any.whl", hash = "sha256:8e17ae5e46d8e7efe51ab6494dd2010f4ca8dae51652aa3c8d55acf50bfb2e99", size = 15095, upload-time = "2025-09-12T07:33:52.639Z" },
]

[[package]]
name = "pytest-mock"
version = "3.15.1"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "pytest" },
]
sdist = { url = "https://files.pythonhosted.org/packages/68/14/eb014d26be205d38ad5ad20d9a80f7d201472e08167f0bb4361e251084a9/pytest_mock-3.15.1.tar.gz", hash = "sha256:1849a238f6f396da19762269de72cb1814ab44416fa73a8686deac10b0d87a0f", size = 34036, upload-time = "2025-09-16T16:37:27.081Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/5a/cc/06253936f4a7fa2e0f48dfe6d851d9c56df896a9ab09ac019d70b760619c/pytest_mock-3.15.1-py3-none-any.whl", hash = "sha256:0a25e2eb88fe5168d535041d09a4529a188176ae608a6d249ee65abc0949630d", size = 10095, upload-time = "2025-09-16T16:37:25.734Z" },
]

[[package]]
name = "pytest-watch"
version = "4.2.0"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "colorama" },
    { name = "docopt" },
    { name = "pytest" },
    { name = "watchdog" },
]
sdist = { url = "https://files.pythonhosted.org/packages/36/47/ab65fc1d682befc318c439940f81a0de1026048479f732e84fe714cd69c0/pytest-watch-4.2.0.tar.gz", hash = "sha256:06136f03d5b361718b8d0d234042f7b2f203910d8568f63df2f866b547b3d4b9", size = 16340, upload-time = "2018-05-20T19:52:16.194Z" }

[[package]]
name = "ruff"
version = "0.13.2"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/02/df/8d7d8c515d33adfc540e2edf6c6021ea1c5a58a678d8cfce9fae59aabcab/ruff-0.13.2.tar.gz", hash = "sha256:cb12fffd32fb16d32cef4ed16d8c7cdc27ed7c944eaa98d99d01ab7ab0b710ff", size = 5416417, upload-time = "2025-09-25T14:54:09.936Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/6e/84/5716a7fa4758e41bf70e603e13637c42cfb9dbf7ceb07180211b9bbf75ef/ruff-0.13.2-py3-none-linux_armv6l.whl", hash = "sha256:3796345842b55f033a78285e4f1641078f902020d8450cade03aad01bffd81c3", size = 12343254, upload-time = "2025-09-25T14:53:27.784Z" },
    { url = "https://files.pythonhosted.org/packages/9b/77/c7042582401bb9ac8eff25360e9335e901d7a1c0749a2b28ba4ecb239991/ruff-0.13.2-py3-none-macosx_10_12_x86_64.whl", hash = "sha256:ff7e4dda12e683e9709ac89e2dd436abf31a4d8a8fc3d89656231ed808e231d2", size = 13040891, upload-time = "2025-09-25T14:53:31.38Z" },
    { url = "https://files.pythonhosted.org/packages/c6/15/125a7f76eb295cb34d19c6778e3a82ace33730ad4e6f28d3427e134a02e0/ruff-0.13.2-py3-none-macosx_11_0_arm64.whl", hash = "sha256:c75e9d2a2fafd1fdd895d0e7e24b44355984affdde1c412a6f6d3f6e16b22d46", size = 12243588, upload-time = "2025-09-25T14:53:33.543Z" },
    { url = "https://files.pythonhosted.org/packages/9e/eb/0093ae04a70f81f8be7fd7ed6456e926b65d238fc122311293d033fdf91e/ruff-0.13.2-py3-none-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:cceac74e7bbc53ed7d15d1042ffe7b6577bf294611ad90393bf9b2a0f0ec7cb6", size = 12491359, upload-time = "2025-09-25T14:53:35.892Z" },
    { url = "https://files.pythonhosted.org/packages/43/fe/72b525948a6956f07dad4a6f122336b6a05f2e3fd27471cea612349fedb9/ruff-0.13.2-py3-none-manylinux_2_17_armv7l.manylinux2014_armv7l.whl", hash = "sha256:6ae3f469b5465ba6d9721383ae9d49310c19b452a161b57507764d7ef15f4b07", size = 12162486, upload-time = "2025-09-25T14:53:38.171Z" },
    { url = "https://files.pythonhosted.org/packages/6a/e3/0fac422bbbfb2ea838023e0d9fcf1f30183d83ab2482800e2cb892d02dfe/ruff-0.13.2-py3-none-manylinux_2_17_i686.manylinux2014_i686.whl", hash = "sha256:4f8f9e3cd6714358238cd6626b9d43026ed19c0c018376ac1ef3c3a04ffb42d8", size = 13871203, upload-time = "2025-09-25T14:53:41.943Z" },
    { url = "https://files.pythonhosted.org/packages/6b/82/b721c8e3ec5df6d83ba0e45dcf00892c4f98b325256c42c38ef136496cbf/ruff-0.13.2-py3-none-manylinux_2_17_ppc64.manylinux2014_ppc64.whl", hash = "sha256:c6ed79584a8f6cbe2e5d7dbacf7cc1ee29cbdb5df1172e77fbdadc8bb85a1f89", size = 14929635, upload-time = "2025-09-25T14:53:43.953Z" },
    { url = "https://files.pythonhosted.org/packages/c4/a0/ad56faf6daa507b83079a1ad7a11694b87d61e6bf01c66bd82b466f21821/ruff-0.13.2-py3-none-manylinux_2_17_ppc64le.manylinux2014_ppc64le.whl", hash = "sha256:aed130b2fde049cea2019f55deb939103123cdd191105f97a0599a3e753d61b0", size = 14338783, upload-time = "2025-09-25T14:53:46.205Z" },
    { url = "https://files.pythonhosted.org/packages/47/77/ad1d9156db8f99cd01ee7e29d74b34050e8075a8438e589121fcd25c4b08/ruff-0.13.2-py3-none-manylinux_2_17_s390x.manylinux2014_s390x.whl", hash = "sha256:1887c230c2c9d65ed1b4e4cfe4d255577ea28b718ae226c348ae68df958191aa", size = 13355322, upload-time = "2025-09-25T14:53:48.164Z" },
    { url = "https://files.pythonhosted.org/packages/64/8b/e87cfca2be6f8b9f41f0bb12dc48c6455e2d66df46fe61bb441a226f1089/ruff-0.13.2-py3-none-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:5bcb10276b69b3cfea3a102ca119ffe5c6ba3901e20e60cf9efb53fa417633c3", size = 13354427, upload-time = "2025-09-25T14:53:50.486Z" },
    { url = "https://files.pythonhosted.org/packages/7f/df/bf382f3fbead082a575edb860897287f42b1b3c694bafa16bc9904c11ed3/ruff-0.13.2-py3-none-manylinux_2_31_riscv64.whl", hash = "sha256:afa721017aa55a555b2ff7944816587f1cb813c2c0a882d158f59b832da1660d", size = 13537637, upload-time = "2025-09-25T14:53:52.887Z" },
    { url = "https://files.pythonhosted.org/packages/51/70/1fb7a7c8a6fc8bd15636288a46e209e81913b87988f26e1913d0851e54f4/ruff-0.13.2-py3-none-musllinux_1_2_aarch64.whl", hash = "sha256:1dbc875cf3720c64b3990fef8939334e74cb0ca65b8dbc61d1f439201a38101b", size = 12340025, upload-time = "2025-09-25T14:53:54.88Z" },
    { url = "https://files.pythonhosted.org/packages/4c/27/1e5b3f1c23ca5dd4106d9d580e5c13d9acb70288bff614b3d7b638378cc9/ruff-0.13.2-py3-none-musllinux_1_2_armv7l.whl", hash = "sha256:5b939a1b2a960e9742e9a347e5bbc9b3c3d2c716f86c6ae273d9cbd64f193f22", size = 12133449, upload-time = "2025-09-25T14:53:57.089Z" },
    { url = "https://files.pythonhosted.org/packages/2d/09/b92a5ccee289f11ab128df57d5911224197d8d55ef3bd2043534ff72ca54/ruff-0.13.2-py3-none-musllinux_1_2_i686.whl", hash = "sha256:50e2d52acb8de3804fc5f6e2fa3ae9bdc6812410a9e46837e673ad1f90a18736", size = 13051369, upload-time = "2025-09-25T14:53:59.124Z" },
    { url = "https://files.pythonhosted.org/packages/89/99/26c9d1c7d8150f45e346dc045cc49f23e961efceb4a70c47dea0960dea9a/ruff-0.13.2-py3-none-musllinux_1_2_x86_64.whl", hash = "sha256:3196bc13ab2110c176b9a4ae5ff7ab676faaa1964b330a1383ba20e1e19645f2", size = 13523644, upload-time = "2025-09-25T14:54:01.622Z" },
    { url = "https://files.pythonhosted.org/packages/f7/00/e7f1501e81e8ec290e79527827af1d88f541d8d26151751b46108978dade/ruff-0.13.2-py3-none-win32.whl", hash = "sha256:7c2a0b7c1e87795fec3404a485096bcd790216c7c146a922d121d8b9c8f1aaac", size = 12245990, upload-time = "2025-09-25T14:54:03.647Z" },
    { url = "https://files.pythonhosted.org/packages/ee/bd/d9f33a73de84fafd0146c6fba4f497c4565fe8fa8b46874b8e438869abc2/ruff-0.13.2-py3-none-win_amd64.whl", hash = "sha256:17d95fb32218357c89355f6f6f9a804133e404fc1f65694372e02a557edf8585", size = 13324004, upload-time = "2025-09-25T14:54:06.05Z" },
    { url = "https://files.pythonhosted.org/packages/c3/12/28fa2f597a605884deb0f65c1b1ae05111051b2a7030f5d8a4ff7f4599ba/ruff-0.13.2-py3-none-win_arm64.whl", hash = "sha256:da711b14c530412c827219312b7d7fbb4877fb31150083add7e8c5336549cea7", size = 12484437, upload-time = "2025-09-25T14:54:08.022Z" },
]

[[package]]
name = "sniffio"
version = "1.3.1"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/a2/87/a6771e1546d97e7e041b6ae58d80074f81b7d5121207425c964ddf5cfdbd/sniffio-1.3.1.tar.gz", hash = "sha256:f4324edc670a0f49750a81b895f35c3adb843cca46f0530f79fc1babb23789dc", size = 20372, upload-time = "2024-02-25T23:20:04.057Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/e9/44/75a9c9421471a6c4805dbf2356f7c181a29c1879239abab1ea2cc8f38b40/sniffio-1.3.1-py3-none-any.whl", hash = "sha256:2f6da418d1f1e0fddd844478f41680e794e6051915791a034ff65e5f100525a2", size = 10235, upload-time = "2024-02-25T23:20:01.196Z" },
]

[[package]]
name = "tomli"
version = "2.2.1"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/18/87/302344fed471e44a87289cf4967697d07e532f2421fdaf868a303cbae4ff/tomli-2.2.1.tar.gz", hash = "sha256:cd45e1dc79c835ce60f7404ec8119f2eb06d38b1deba146f07ced3bbc44505ff", size = 17175, upload-time = "2024-11-27T22:38:36.873Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/43/ca/75707e6efa2b37c77dadb324ae7d9571cb424e61ea73fad7c56c2d14527f/tomli-2.2.1-cp311-cp311-macosx_10_9_x86_64.whl", hash = "sha256:678e4fa69e4575eb77d103de3df8a895e1591b48e740211bd1067378c69e8249", size = 131077, upload-time = "2024-11-27T22:37:54.956Z" },
    { url = "https://files.pythonhosted.org/packages/c7/16/51ae563a8615d472fdbffc43a3f3d46588c264ac4f024f63f01283becfbb/tomli-2.2.1-cp311-cp311-macosx_11_0_arm64.whl", hash = "sha256:023aa114dd824ade0100497eb2318602af309e5a55595f76b626d6d9f3b7b0a6", size = 123429, upload-time = "2024-11-27T22:37:56.698Z" },
    { url = "https://files.pythonhosted.org/packages/f1/dd/4f6cd1e7b160041db83c694abc78e100473c15d54620083dbd5aae7b990e/tomli-2.2.1-cp311-cp311-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:ece47d672db52ac607a3d9599a9d48dcb2f2f735c6c2d1f34130085bb12b112a", size = 226067, upload-time = "2024-11-27T22:37:57.63Z" },
    { url = "https://files.pythonhosted.org/packages/a9/6b/c54ede5dc70d648cc6361eaf429304b02f2871a345bbdd51e993d6cdf550/tomli-2.2.1-cp311-cp311-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:6972ca9c9cc9f0acaa56a8ca1ff51e7af152a9f87fb64623e31d5c83700080ee", size = 236030, upload-time = "2024-11-27T22:37:59.344Z" },
    { url = "https://files.pythonhosted.org/packages/1f/47/999514fa49cfaf7a92c805a86c3c43f4215621855d151b61c602abb38091/tomli-2.2.1-cp311-cp311-manylinux_2_5_i686.manylinux1_i686.manylinux_2_17_i686.manylinux2014_i686.whl", hash = "sha256:c954d2250168d28797dd4e3ac5cf812a406cd5a92674ee4c8f123c889786aa8e", size = 240898, upload-time = "2024-11-27T22:38:00.429Z" },
    { url = "https://files.pythonhosted.org/packages/73/41/0a01279a7ae09ee1573b423318e7934674ce06eb33f50936655071d81a24/tomli-2.2.1-cp311-cp311-musllinux_1_2_aarch64.whl", hash = "sha256:8dd28b3e155b80f4d54beb40a441d366adcfe740969820caf156c019fb5c7ec4", size = 229894, upload-time = "2024-11-27T22:38:02.094Z" },
    { url = "https://files.pythonhosted.org/packages/55/18/5d8bc5b0a0362311ce4d18830a5d28943667599a60d20118074ea1b01bb7/tomli-2.2.1-cp311-cp311-musllinux_1_2_i686.whl", hash = "sha256:e59e304978767a54663af13c07b3d1af22ddee3bb2fb0618ca1593e4f593a106", size = 245319, upload-time = "2024-11-27T22:38:03.206Z" },
    { url = "https://files.pythonhosted.org/packages/92/a3/7ade0576d17f3cdf5ff44d61390d4b3febb8a9fc2b480c75c47ea048c646/tomli-2.2.1-cp311-cp311-musllinux_1_2_x86_64.whl", hash = "sha256:33580bccab0338d00994d7f16f4c4ec25b776af3ffaac1ed74e0b3fc95e885a8", size = 238273, upload-time = "2024-11-27T22:38:04.217Z" },
    { url = "https://files.pythonhosted.org/packages/72/6f/fa64ef058ac1446a1e51110c375339b3ec6be245af9d14c87c4a6412dd32/tomli-2.2.1-cp311-cp311-win32.whl", hash = "sha256:465af0e0875402f1d226519c9904f37254b3045fc5084697cefb9bdde1ff99ff", size = 98310, upload-time = "2024-11-27T22:38:05.908Z" },
    { url = "https://files.pythonhosted.org/packages/6a/1c/4a2dcde4a51b81be3530565e92eda625d94dafb46dbeb15069df4caffc34/tomli-2.2.1-cp311-cp311-win_amd64.whl", hash = "sha256:2d0f2fdd22b02c6d81637a3c95f8cd77f995846af7414c5c4b8d0545afa1bc4b", size = 108309, upload-time = "2024-11-27T22:38:06.812Z" },
    { url = "https://files.pythonhosted.org/packages/52/e1/f8af4c2fcde17500422858155aeb0d7e93477a0d59a98e56cbfe75070fd0/tomli-2.2.1-cp312-cp312-macosx_10_13_x86_64.whl", hash = "sha256:4a8f6e44de52d5e6c657c9fe83b562f5f4256d8ebbfe4ff922c495620a7f6cea", size = 132762, upload-time = "2024-11-27T22:38:07.731Z" },
    { url = "https://files.pythonhosted.org/packages/03/b8/152c68bb84fc00396b83e7bbddd5ec0bd3dd409db4195e2a9b3e398ad2e3/tomli-2.2.1-cp312-cp312-macosx_11_0_arm64.whl", hash = "sha256:8d57ca8095a641b8237d5b079147646153d22552f1c637fd3ba7f4b0b29167a8", size = 123453, upload-time = "2024-11-27T22:38:09.384Z" },
    { url = "https://files.pythonhosted.org/packages/c8/d6/fc9267af9166f79ac528ff7e8c55c8181ded34eb4b0e93daa767b8841573/tomli-2.2.1-cp312-cp312-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:4e340144ad7ae1533cb897d406382b4b6fede8890a03738ff1683af800d54192", size = 233486, upload-time = "2024-11-27T22:38:10.329Z" },
    { url = "https://files.pythonhosted.org/packages/5c/51/51c3f2884d7bab89af25f678447ea7d297b53b5a3b5730a7cb2ef6069f07/tomli-2.2.1-cp312-cp312-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:db2b95f9de79181805df90bedc5a5ab4c165e6ec3fe99f970d0e302f384ad222", size = 242349, upload-time = "2024-11-27T22:38:11.443Z" },
    { url = "https://files.pythonhosted.org/packages/ab/df/bfa89627d13a5cc22402e441e8a931ef2108403db390ff3345c05253935e/tomli-2.2.1-cp312-cp312-manylinux_2_5_i686.manylinux1_i686.manylinux_2_17_i686.manylinux2014_i686.whl", hash = "sha256:40741994320b232529c802f8bc86da4e1aa9f413db394617b9a256ae0f9a7f77", size = 252159, upload-time = "2024-11-27T22:38:13.099Z" },
    { url = "https://files.pythonhosted.org/packages/9e/6e/fa2b916dced65763a5168c6ccb91066f7639bdc88b48adda990db10c8c0b/tomli-2.2.1-cp312-cp312-musllinux_1_2_aarch64.whl", hash = "sha256:400e720fe168c0f8521520190686ef8ef033fb19fc493da09779e592861b78c6", size = 237243, upload-time = "2024-11-27T22:38:14.766Z" },
    { url = "https://files.pythonhosted.org/packages/b4/04/885d3b1f650e1153cbb93a6a9782c58a972b94ea4483ae4ac5cedd5e4a09/tomli-2.2.1-cp312-cp312-musllinux_1_2_i686.whl", hash = "sha256:02abe224de6ae62c19f090f68da4e27b10af2b93213d36cf44e6e1c5abd19fdd", size = 259645, upload-time = "2024-11-27T22:38:15.843Z" },
    { url = "https://files.pythonhosted.org/packages/9c/de/6b432d66e986e501586da298e28ebeefd3edc2c780f3ad73d22566034239/tomli-2.2.1-cp312-cp312-musllinux_1_2_x86_64.whl", hash = "sha256:b82ebccc8c8a36f2094e969560a1b836758481f3dc360ce9a3277c65f374285e", size = 244584, upload-time = "2024-11-27T22:38:17.645Z" },
    { url = "https://files.pythonhosted.org/packages/1c/9a/47c0449b98e6e7d1be6cbac02f93dd79003234ddc4aaab6ba07a9a7482e2/tomli-2.2.1-cp312-cp312-win32.whl", hash = "sha256:889f80ef92701b9dbb224e49ec87c645ce5df3fa2cc548664eb8a25e03127a98", size = 98875, upload-time = "2024-11-27T22:38:19.159Z" },
    { url = "https://files.pythonhosted.org/packages/ef/60/9b9638f081c6f1261e2688bd487625cd1e660d0a85bd469e91d8db969734/tomli-2.2.1-cp312-cp312-win_amd64.whl", hash = "sha256:7fc04e92e1d624a4a63c76474610238576942d6b8950a2d7f908a340494e67e4", size = 109418, upload-time = "2024-11-27T22:38:20.064Z" },
    { url = "https://files.pythonhosted.org/packages/04/90/2ee5f2e0362cb8a0b6499dc44f4d7d48f8fff06d28ba46e6f1eaa61a1388/tomli-2.2.1-cp313-cp313-macosx_10_13_x86_64.whl", hash = "sha256:f4039b9cbc3048b2416cc57ab3bda989a6fcf9b36cf8937f01a6e731b64f80d7", size = 132708, upload-time = "2024-11-27T22:38:21.659Z" },
    { url = "https://files.pythonhosted.org/packages/c0/ec/46b4108816de6b385141f082ba99e315501ccd0a2ea23db4a100dd3990ea/tomli-2.2.1-cp313-cp313-macosx_11_0_arm64.whl", hash = "sha256:286f0ca2ffeeb5b9bd4fcc8d6c330534323ec51b2f52da063b11c502da16f30c", size = 123582, upload-time = "2024-11-27T22:38:22.693Z" },
    { url = "https://files.pythonhosted.org/packages/a0/bd/b470466d0137b37b68d24556c38a0cc819e8febe392d5b199dcd7f578365/tomli-2.2.1-cp313-cp313-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:a92ef1a44547e894e2a17d24e7557a5e85a9e1d0048b0b5e7541f76c5032cb13", size = 232543, upload-time = "2024-11-27T22:38:24.367Z" },
    { url = "https://files.pythonhosted.org/packages/d9/e5/82e80ff3b751373f7cead2815bcbe2d51c895b3c990686741a8e56ec42ab/tomli-2.2.1-cp313-cp313-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:9316dc65bed1684c9a98ee68759ceaed29d229e985297003e494aa825ebb0281", size = 241691, upload-time = "2024-11-27T22:38:26.081Z" },
    { url = "https://files.pythonhosted.org/packages/05/7e/2a110bc2713557d6a1bfb06af23dd01e7dde52b6ee7dadc589868f9abfac/tomli-2.2.1-cp313-cp313-manylinux_2_5_i686.manylinux1_i686.manylinux_2_17_i686.manylinux2014_i686.whl", hash = "sha256:e85e99945e688e32d5a35c1ff38ed0b3f41f43fad8df0bdf79f72b2ba7bc5272", size = 251170, upload-time = "2024-11-27T22:38:27.921Z" },
    { url = "https://files.pythonhosted.org/packages/64/7b/22d713946efe00e0adbcdfd6d1aa119ae03fd0b60ebed51ebb3fa9f5a2e5/tomli-2.2.1-cp313-cp313-musllinux_1_2_aarch64.whl", hash = "sha256:ac065718db92ca818f8d6141b5f66369833d4a80a9d74435a268c52bdfa73140", size = 236530, upload-time = "2024-11-27T22:38:29.591Z" },
    { url = "https://files.pythonhosted.org/packages/38/31/3a76f67da4b0cf37b742ca76beaf819dca0ebef26d78fc794a576e08accf/tomli-2.2.1-cp313-cp313-musllinux_1_2_i686.whl", hash = "sha256:d920f33822747519673ee656a4b6ac33e382eca9d331c87770faa3eef562aeb2", size = 258666, upload-time = "2024-11-27T22:38:30.639Z" },
    { url = "https://files.pythonhosted.org/packages/07/10/5af1293da642aded87e8a988753945d0cf7e00a9452d3911dd3bb354c9e2/tomli-2.2.1-cp313-cp313-musllinux_1_2_x86_64.whl", hash = "sha256:a198f10c4d1b1375d7687bc25294306e551bf1abfa4eace6650070a5c1ae2744", size = 243954, upload-time = "2024-11-27T22:38:31.702Z" },
    { url = "https://files.pythonhosted.org/packages/5b/b9/1ed31d167be802da0fc95020d04cd27b7d7065cc6fbefdd2f9186f60d7bd/tomli-2.2.1-cp313-cp313-win32.whl", hash = "sha256:d3f5614314d758649ab2ab3a62d4f2004c825922f9e370b29416484086b264ec", size = 98724, upload-time = "2024-11-27T22:38:32.837Z" },
    { url = "https://files.pythonhosted.org/packages/c7/32/b0963458706accd9afcfeb867c0f9175a741bf7b19cd424230714d722198/tomli-2.2.1-cp313-cp313-win_amd64.whl", hash = "sha256:a38aa0308e754b0e3c67e344754dff64999ff9b513e691d0e786265c93583c69", size = 109383, upload-time = "2024-11-27T22:38:34.455Z" },
    { url = "https://files.pythonhosted.org/packages/6e/c2/61d3e0f47e2b74ef40a68b9e6ad5984f6241a942f7cd3bbfbdbd03861ea9/tomli-2.2.1-py3-none-any.whl", hash = "sha256:cb55c73c5f4408779d0cf3eef9f762b9c9f147a77de7b258bef0a5628adc85cc", size = 14257, upload-time = "2024-11-27T22:38:35.385Z" },
]

[[package]]
name = "typing-extensions"
version = "4.15.0"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/72/94/1a15dd82efb362ac84269196e94cf00f187f7ed21c242792a923cdb1c61f/typing_extensions-4.15.0.tar.gz", hash = "sha256:0cea48d173cc12fa28ecabc3b837ea3cf6f38c6d1136f85cbaaf598984861466", size = 109391, upload-time = "2025-08-25T13:49:26.313Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/18/67/36e9267722cc04a6b9f15c7f3441c2363321a3ea07da7ae0c0707beb2a9c/typing_extensions-4.15.0-py3-none-any.whl", hash = "sha256:f0fa19c6845758ab08074a0cfa8b7aecb71c999ca73d62883bc25cc018c4e548", size = 44614, upload-time = "2025-08-25T13:49:24.86Z" },
]

[[package]]
name = "watchdog"
version = "6.0.0"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/db/7d/7f3d619e951c88ed75c6037b246ddcf2d322812ee8ea189be89511721d54/watchdog-6.0.0.tar.gz", hash = "sha256:9ddf7c82fda3ae8e24decda1338ede66e1c99883db93711d8fb941eaa2d8c282", size = 131220, upload-time = "2024-11-01T14:07:13.037Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/0c/56/90994d789c61df619bfc5ce2ecdabd5eeff564e1eb47512bd01b5e019569/watchdog-6.0.0-cp310-cp310-macosx_10_9_universal2.whl", hash = "sha256:d1cdb490583ebd691c012b3d6dae011000fe42edb7a82ece80965b42abd61f26", size = 96390, upload-time = "2024-11-01T14:06:24.793Z" },
    { url = "https://files.pythonhosted.org/packages/55/46/9a67ee697342ddf3c6daa97e3a587a56d6c4052f881ed926a849fcf7371c/watchdog-6.0.0-cp310-cp310-macosx_10_9_x86_64.whl", hash = "sha256:bc64ab3bdb6a04d69d4023b29422170b74681784ffb9463ed4870cf2f3e66112", size = 88389, upload-time = "2024-11-01T14:06:27.112Z" },
    { url = "https://files.pythonhosted.org/packages/44/65/91b0985747c52064d8701e1075eb96f8c40a79df889e59a399453adfb882/watchdog-6.0.0-cp310-cp310-macosx_11_0_arm64.whl", hash = "sha256:c897ac1b55c5a1461e16dae288d22bb2e412ba9807df8397a635d88f671d36c3", size = 89020, upload-time = "2024-11-01T14:06:29.876Z" },
    { url = "https://files.pythonhosted.org/packages/e0/24/d9be5cd6642a6aa68352ded4b4b10fb0d7889cb7f45814fb92cecd35f101/watchdog-6.0.0-cp311-cp311-macosx_10_9_universal2.whl", hash = "sha256:6eb11feb5a0d452ee41f824e271ca311a09e250441c262ca2fd7ebcf2461a06c", size = 96393, upload-time = "2024-11-01T14:06:31.756Z" },
    { url = "https://files.pythonhosted.org/packages/63/7a/6013b0d8dbc56adca7fdd4f0beed381c59f6752341b12fa0886fa7afc78b/watchdog-6.0.0-cp311-cp311-macosx_10_9_x86_64.whl", hash = "sha256:ef810fbf7b781a5a593894e4f439773830bdecb885e6880d957d5b9382a960d2", size = 88392, upload-time = "2024-11-01T14:06:32.99Z" },
    { url = "https://files.pythonhosted.org/packages/d1/40/b75381494851556de56281e053700e46bff5b37bf4c7267e858640af5a7f/watchdog-6.0.0-cp311-cp311-macosx_11_0_arm64.whl", hash = "sha256:afd0fe1b2270917c5e23c2a65ce50c2a4abb63daafb0d419fde368e272a76b7c", size = 89019, upload-time = "2024-11-01T14:06:34.963Z" },
    { url = "https://files.pythonhosted.org/packages/39/ea/3930d07dafc9e286ed356a679aa02d777c06e9bfd1164fa7c19c288a5483/watchdog-6.0.0-cp312-cp312-macosx_10_13_universal2.whl", hash = "sha256:bdd4e6f14b8b18c334febb9c4425a878a2ac20efd1e0b231978e7b150f92a948", size = 96471, upload-time = "2024-11-01T14:06:37.745Z" },
    { url = "https://files.pythonhosted.org/packages/12/87/48361531f70b1f87928b045df868a9fd4e253d9ae087fa4cf3f7113be363/watchdog-6.0.0-cp312-cp312-macosx_10_13_x86_64.whl", hash = "sha256:c7c15dda13c4eb00d6fb6fc508b3c0ed88b9d5d374056b239c4ad1611125c860", size = 88449, upload-time = "2024-11-01T14:06:39.748Z" },
    { url = "https://files.pythonhosted.org/packages/5b/7e/8f322f5e600812e6f9a31b75d242631068ca8f4ef0582dd3ae6e72daecc8/watchdog-6.0.0-cp312-cp312-macosx_11_0_arm64.whl", hash = "sha256:6f10cb2d5902447c7d0da897e2c6768bca89174d0c6e1e30abec5421af97a5b0", size = 89054, upload-time = "2024-11-01T14:06:41.009Z" },
    { url = "https://files.pythonhosted.org/packages/68/98/b0345cabdce2041a01293ba483333582891a3bd5769b08eceb0d406056ef/watchdog-6.0.0-cp313-cp313-macosx_10_13_universal2.whl", hash = "sha256:490ab2ef84f11129844c23fb14ecf30ef3d8a6abafd3754a6f75ca1e6654136c", size = 96480, upload-time = "2024-11-01T14:06:42.952Z" },
    { url = "https://files.pythonhosted.org/packages/85/83/cdf13902c626b28eedef7ec4f10745c52aad8a8fe7eb04ed7b1f111ca20e/watchdog-6.0.0-cp313-cp313-macosx_10_13_x86_64.whl", hash = "sha256:76aae96b00ae814b181bb25b1b98076d5fc84e8a53cd8885a318b42b6d3a5134", size = 88451, upload-time = "2024-11-01T14:06:45.084Z" },
    { url = "https://files.pythonhosted.org/packages/fe/c4/225c87bae08c8b9ec99030cd48ae9c4eca050a59bf5c2255853e18c87b50/watchdog-6.0.0-cp313-cp313-macosx_11_0_arm64.whl", hash = "sha256:a175f755fc2279e0b7312c0035d52e27211a5bc39719dd529625b1930917345b", size = 89057, upload-time = "2024-11-01T14:06:47.324Z" },
    { url = "https://files.pythonhosted.org/packages/30/ad/d17b5d42e28a8b91f8ed01cb949da092827afb9995d4559fd448d0472763/watchdog-6.0.0-pp310-pypy310_pp73-macosx_10_15_x86_64.whl", hash = "sha256:c7ac31a19f4545dd92fc25d200694098f42c9a8e391bc00bdd362c5736dbf881", size = 87902, upload-time = "2024-11-01T14:06:53.119Z" },
    { url = "https://files.pythonhosted.org/packages/5c/ca/c3649991d140ff6ab67bfc85ab42b165ead119c9e12211e08089d763ece5/watchdog-6.0.0-pp310-pypy310_pp73-macosx_11_0_arm64.whl", hash = "sha256:9513f27a1a582d9808cf21a07dae516f0fab1cf2d7683a742c498b93eedabb11", size = 88380, upload-time = "2024-11-01T14:06:55.19Z" },
    { url = "https://files.pythonhosted.org/packages/a9/c7/ca4bf3e518cb57a686b2feb4f55a1892fd9a3dd13f470fca14e00f80ea36/watchdog-6.0.0-py3-none-manylinux2014_aarch64.whl", hash = "sha256:7607498efa04a3542ae3e05e64da8202e58159aa1fa4acddf7678d34a35d4f13", size = 79079, upload-time = "2024-11-01T14:06:59.472Z" },
    { url = "https://files.pythonhosted.org/packages/5c/51/d46dc9332f9a647593c947b4b88e2381c8dfc0942d15b8edc0310fa4abb1/watchdog-6.0.0-py3-none-manylinux2014_armv7l.whl", hash = "sha256:9041567ee8953024c83343288ccc458fd0a2d811d6a0fd68c4c22609e3490379", size = 79078, upload-time = "2024-11-01T14:07:01.431Z" },
    { url = "https://files.pythonhosted.org/packages/d4/57/04edbf5e169cd318d5f07b4766fee38e825d64b6913ca157ca32d1a42267/watchdog-6.0.0-py3-none-manylinux2014_i686.whl", hash = "sha256:82dc3e3143c7e38ec49d61af98d6558288c415eac98486a5c581726e0737c00e", size = 79076, upload-time = "2024-11-01T14:07:02.568Z" },
    { url = "https://files.pythonhosted.org/packages/ab/cc/da8422b300e13cb187d2203f20b9253e91058aaf7db65b74142013478e66/watchdog-6.0.0-py3-none-manylinux2014_ppc64.whl", hash = "sha256:212ac9b8bf1161dc91bd09c048048a95ca3a4c4f5e5d4a7d1b1a7d5752a7f96f", size = 79077, upload-time = "2024-11-01T14:07:03.893Z" },
    { url = "https://files.pythonhosted.org/packages/2c/3b/b8964e04ae1a025c44ba8e4291f86e97fac443bca31de8bd98d3263d2fcf/watchdog-6.0.0-py3-none-manylinux2014_ppc64le.whl", hash = "sha256:e3df4cbb9a450c6d49318f6d14f4bbc80d763fa587ba46ec86f99f9e6876bb26", size = 79078, upload-time = "2024-11-01T14:07:05.189Z" },
    { url = "https://files.pythonhosted.org/packages/62/ae/a696eb424bedff7407801c257d4b1afda455fe40821a2be430e173660e81/watchdog-6.0.0-py3-none-manylinux2014_s390x.whl", hash = "sha256:2cce7cfc2008eb51feb6aab51251fd79b85d9894e98ba847408f662b3395ca3c", size = 79077, upload-time = "2024-11-01T14:07:06.376Z" },
    { url = "https://files.pythonhosted.org/packages/b5/e8/dbf020b4d98251a9860752a094d09a65e1b436ad181faf929983f697048f/watchdog-6.0.0-py3-none-manylinux2014_x86_64.whl", hash = "sha256:20ffe5b202af80ab4266dcd3e91aae72bf2da48c0d33bdb15c66658e685e94e2", size = 79078, upload-time = "2024-11-01T14:07:07.547Z" },
    { url = "https://files.pythonhosted.org/packages/07/f6/d0e5b343768e8bcb4cda79f0f2f55051bf26177ecd5651f84c07567461cf/watchdog-6.0.0-py3-none-win32.whl", hash = "sha256:07df1fdd701c5d4c8e55ef6cf55b8f0120fe1aef7ef39a1c6fc6bc2e606d517a", size = 79065, upload-time = "2024-11-01T14:07:09.525Z" },
    { url = "https://files.pythonhosted.org/packages/db/d9/c495884c6e548fce18a8f40568ff120bc3a4b7b99813081c8ac0c936fa64/watchdog-6.0.0-py3-none-win_amd64.whl", hash = "sha256:cbafb470cf848d93b5d013e2ecb245d4aa1c8fd0504e863ccefa32445359d680", size = 79070, upload-time = "2024-11-01T14:07:10.686Z" },
    { url = "https://files.pythonhosted.org/packages/33/e8/e40370e6d74ddba47f002a32919d91310d6074130fe4e17dabcafc15cbf1/watchdog-6.0.0-py3-none-win_ia64.whl", hash = "sha256:a1914259fa9e1454315171103c6a30961236f508b9b623eae470268bbcc6a22f", size = 79067, upload-time = "2024-11-01T14:07:11.845Z" },
]

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/sdk-py/langgraph_sdk/__init__.py
```py
from langgraph_sdk.auth import Auth
from langgraph_sdk.client import get_client, get_sync_client

__version__ = "0.2.9"

__all__ = ["Auth", "get_client", "get_sync_client"]

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/sdk-py/langgraph_sdk/client.py
```py
"""The LangGraph client implementations connect to the LangGraph API.

This module provides both asynchronous (`get_client(url="http://localhost:2024")` or
`LangGraphClient`) and synchronous (`get_sync_client(url="http://localhost:2024")` or
`SyncLanggraphClient`) clients to interacting with the LangGraph API's core resources
such as Assistants, Threads, Runs, and Cron jobs, as well as its persistent document
Store.
"""

from __future__ import annotations

import asyncio
import functools
import logging
import os
import re
import sys
import warnings
from collections.abc import AsyncIterator, Callable, Iterator, Mapping, Sequence
from types import TracebackType
from typing import (
    Any,
    Literal,
    overload,
)

import httpx
import orjson

import langgraph_sdk
from langgraph_sdk.errors import _araise_for_status_typed, _raise_for_status_typed
from langgraph_sdk.schema import (
    All,
    Assistant,
    AssistantSelectField,
    AssistantSortBy,
    AssistantVersion,
    CancelAction,
    Checkpoint,
    Command,
    Config,
    Context,
    Cron,
    CronSelectField,
    CronSortBy,
    DisconnectMode,
    Durability,
    GraphSchema,
    IfNotExists,
    Item,
    Json,
    ListNamespaceResponse,
    MultitaskStrategy,
    OnCompletionBehavior,
    OnConflictBehavior,
    QueryParamTypes,
    Run,
    RunCreate,
    RunCreateMetadata,
    RunSelectField,
    RunStatus,
    SearchItemsResponse,
    SortOrder,
    StreamMode,
    StreamPart,
    Subgraphs,
    Thread,
    ThreadSelectField,
    ThreadSortBy,
    ThreadState,
    ThreadStatus,
    ThreadStreamMode,
    ThreadUpdateStateResponse,
)
from langgraph_sdk.sse import SSEDecoder, aiter_lines_raw, iter_lines_raw

logger = logging.getLogger(__name__)


RESERVED_HEADERS = ("x-api-key",)


def _get_api_key(api_key: str | None = None) -> str | None:
    """Get the API key from the environment.
    Precedence:
        1. explicit argument
        2. LANGGRAPH_API_KEY
        3. LANGSMITH_API_KEY
        4. LANGCHAIN_API_KEY
    """
    if api_key:
        return api_key
    for prefix in ["LANGGRAPH", "LANGSMITH", "LANGCHAIN"]:
        if env := os.getenv(f"{prefix}_API_KEY"):
            return env.strip().strip('"').strip("'")
    return None  # type: ignore


def _get_headers(
    api_key: str | None, custom_headers: Mapping[str, str] | None
) -> dict[str, str]:
    """Combine api_key and custom user-provided headers."""
    custom_headers = custom_headers or {}
    for header in RESERVED_HEADERS:
        if header in custom_headers:
            raise ValueError(f"Cannot set reserved header '{header}'")

    headers = {
        "User-Agent": f"langgraph-sdk-py/{langgraph_sdk.__version__}",
        **custom_headers,
    }
    api_key = _get_api_key(api_key)
    if api_key:
        headers["x-api-key"] = api_key

    return headers


def _orjson_default(obj: Any) -> Any:
    if hasattr(obj, "model_dump") and callable(obj.model_dump):
        return obj.model_dump()
    elif hasattr(obj, "dict") and callable(obj.dict):
        return obj.dict()
    elif isinstance(obj, (set, frozenset)):
        return list(obj)
    else:
        raise TypeError(f"Object of type {type(obj)} is not JSON serializable")


# Compiled regex pattern for extracting run metadata from Content-Location header
_RUN_METADATA_PATTERN = re.compile(
    r"(\/threads\/(?P<thread_id>.+))?\/runs\/(?P<run_id>.+)"
)


def _get_run_metadata_from_response(
    response: httpx.Response,
) -> RunCreateMetadata | None:
    """Extract run metadata from the response headers."""
    if (content_location := response.headers.get("Content-Location")) and (
        match := _RUN_METADATA_PATTERN.search(content_location)
    ):
        return RunCreateMetadata(
            run_id=match.group("run_id"),
            thread_id=match.group("thread_id") or None,
        )

    return None


def get_client(
    *,
    url: str | None = None,
    api_key: str | None = None,
    headers: Mapping[str, str] | None = None,
    timeout: TimeoutTypes | None = None,
) -> LangGraphClient:
    """Create and configure a LangGraphClient.

    The client provides programmatic access to LangSmith Deployment. It supports
    both remote servers and local in-process connections (when running inside a LangGraph server).

    Args:
        url:
            Base URL of the LangGraph API.
            – If `None`, the client first attempts an in-process connection via ASGI transport.
              If that fails, it falls back to `http://localhost:8123`.
        api_key:
            API key for authentication. If omitted, the client reads from environment
            variables in the following order:
              1. Function argument
              2. `LANGGRAPH_API_KEY`
              3. `LANGSMITH_API_KEY`
              4. `LANGCHAIN_API_KEY`
        headers:
            Additional HTTP headers to include in requests. Merged with authentication headers.
        timeout:
            HTTP timeout configuration. May be:
              – `httpx.Timeout` instance
              – float (total seconds)
              – tuple `(connect, read, write, pool)` in seconds
            Defaults: connect=5, read=300, write=300, pool=5.

    Returns:
        LangGraphClient:
            A top-level client exposing sub-clients for assistants, threads,
            runs, and cron operations.

    ???+ example "Connect to a remote server:"

        ```python
        from langgraph_sdk import get_client

        # get top-level LangGraphClient
        client = get_client(url="http://localhost:8123")

        # example usage: client.<model>.<method_name>()
        assistants = await client.assistants.get(assistant_id="some_uuid")
        ```

    ???+ example "Connect in-process to a running LangGraph server:"

        ```python
        from langgraph_sdk import get_client

        client = get_client(url=None)

        async def my_node(...):
            subagent_result = await client.runs.wait(
                thread_id=None,
                assistant_id="agent",
                input={"messages": [{"role": "user", "content": "Foo"}]},
            )
        ```
    """

    transport: httpx.AsyncBaseTransport | None = None
    if url is None:
        if os.environ.get("__LANGGRAPH_DEFER_LOOPBACK_TRANSPORT") == "true":
            transport = get_asgi_transport()(app=None, root_path="/noauth")
            _registered_transports.append(transport)
            url = "http://api"
        else:
            try:
                from langgraph_api.server import app  # type: ignore

                url = "http://api"

                transport = get_asgi_transport()(app, root_path="/noauth")
            except Exception:
                url = "http://localhost:8123"

    if transport is None:
        transport = httpx.AsyncHTTPTransport(retries=5)
    client = httpx.AsyncClient(
        base_url=url,
        transport=transport,
        timeout=(
            httpx.Timeout(timeout)
            if timeout is not None
            else httpx.Timeout(connect=5, read=300, write=300, pool=5)
        ),
        headers=_get_headers(api_key, headers),
    )
    return LangGraphClient(client)


class LangGraphClient:
    """Top-level client for LangGraph API.

    Attributes:
        assistants: Manages versioned configuration for your graphs.
        threads: Handles (potentially) multi-turn interactions, such as conversational threads.
        runs: Controls individual invocations of the graph.
        crons: Manages scheduled operations.
        store: Interfaces with persistent, shared data storage.
    """

    def __init__(self, client: httpx.AsyncClient) -> None:
        self.http = HttpClient(client)
        self.assistants = AssistantsClient(self.http)
        self.threads = ThreadsClient(self.http)
        self.runs = RunsClient(self.http)
        self.crons = CronClient(self.http)
        self.store = StoreClient(self.http)

    async def __aenter__(self) -> LangGraphClient:
        """Enter the async context manager."""
        return self

    async def __aexit__(
        self,
        exc_type: type[BaseException] | None,
        exc_val: BaseException | None,
        exc_tb: TracebackType | None,
    ) -> None:
        """Exit the async context manager."""
        await self.aclose()

    async def aclose(self) -> None:
        """Close the underlying HTTP client."""
        if hasattr(self, "http"):
            await self.http.client.aclose()


class HttpClient:
    """Handle async requests to the LangGraph API.

    Adds additional error messaging & content handling above the
    provided httpx client.

    Attributes:
        client (httpx.AsyncClient): Underlying HTTPX async client.
    """

    def __init__(self, client: httpx.AsyncClient) -> None:
        self.client = client

    async def get(
        self,
        path: str,
        *,
        params: QueryParamTypes | None = None,
        headers: Mapping[str, str] | None = None,
        on_response: Callable[[httpx.Response], None] | None = None,
    ) -> Any:
        """Send a `GET` request."""
        r = await self.client.get(path, params=params, headers=headers)
        if on_response:
            on_response(r)
        await _araise_for_status_typed(r)
        return await _adecode_json(r)

    async def post(
        self,
        path: str,
        *,
        json: dict[str, Any] | list | None,
        params: QueryParamTypes | None = None,
        headers: Mapping[str, str] | None = None,
        on_response: Callable[[httpx.Response], None] | None = None,
    ) -> Any:
        """Send a `POST` request."""
        if json is not None:
            request_headers, content = await _aencode_json(json)
        else:
            request_headers, content = {}, b""
        # Merge headers, with runtime headers taking precedence
        if headers:
            request_headers.update(headers)
        r = await self.client.post(
            path, headers=request_headers, content=content, params=params
        )
        if on_response:
            on_response(r)
        await _araise_for_status_typed(r)
        return await _adecode_json(r)

    async def put(
        self,
        path: str,
        *,
        json: dict,
        params: QueryParamTypes | None = None,
        headers: Mapping[str, str] | None = None,
        on_response: Callable[[httpx.Response], None] | None = None,
    ) -> Any:
        """Send a `PUT` request."""
        request_headers, content = await _aencode_json(json)
        if headers:
            request_headers.update(headers)
        r = await self.client.put(
            path, headers=request_headers, content=content, params=params
        )
        if on_response:
            on_response(r)
        await _araise_for_status_typed(r)
        return await _adecode_json(r)

    async def patch(
        self,
        path: str,
        *,
        json: dict,
        params: QueryParamTypes | None = None,
        headers: Mapping[str, str] | None = None,
        on_response: Callable[[httpx.Response], None] | None = None,
    ) -> Any:
        """Send a `PATCH` request."""
        request_headers, content = await _aencode_json(json)
        if headers:
            request_headers.update(headers)
        r = await self.client.patch(
            path, headers=request_headers, content=content, params=params
        )
        if on_response:
            on_response(r)
        await _araise_for_status_typed(r)
        return await _adecode_json(r)

    async def delete(
        self,
        path: str,
        *,
        json: Any | None = None,
        params: QueryParamTypes | None = None,
        headers: Mapping[str, str] | None = None,
        on_response: Callable[[httpx.Response], None] | None = None,
    ) -> None:
        """Send a `DELETE` request."""
        r = await self.client.request(
            "DELETE", path, json=json, params=params, headers=headers
        )
        if on_response:
            on_response(r)
        await _araise_for_status_typed(r)

    async def request_reconnect(
        self,
        path: str,
        method: str,
        *,
        json: dict[str, Any] | None = None,
        params: QueryParamTypes | None = None,
        headers: Mapping[str, str] | None = None,
        on_response: Callable[[httpx.Response], None] | None = None,
        reconnect_limit: int = 5,
    ) -> Any:
        """Send a request that automatically reconnects to Location header."""
        request_headers, content = await _aencode_json(json)
        if headers:
            request_headers.update(headers)
        async with self.client.stream(
            method, path, headers=request_headers, content=content, params=params
        ) as r:
            if on_response:
                on_response(r)
            try:
                r.raise_for_status()
            except httpx.HTTPStatusError as e:
                body = (await r.aread()).decode()
                if sys.version_info >= (3, 11):
                    e.add_note(body)
                else:
                    logger.error(f"Error from langgraph-api: {body}", exc_info=e)
                raise e
            loc = r.headers.get("location")
            if reconnect_limit <= 0 or not loc:
                return await _adecode_json(r)
            try:
                return await _adecode_json(r)
            except httpx.HTTPError:
                warnings.warn(
                    f"Request failed, attempting reconnect to Location: {loc}",
                    stacklevel=2,
                )
                await r.aclose()
                return await self.request_reconnect(
                    loc,
                    "GET",
                    headers=request_headers,
                    # don't pass on_response so it's only called once
                    reconnect_limit=reconnect_limit - 1,
                )

    async def stream(
        self,
        path: str,
        method: str,
        *,
        json: dict[str, Any] | None = None,
        params: QueryParamTypes | None = None,
        headers: Mapping[str, str] | None = None,
        on_response: Callable[[httpx.Response], None] | None = None,
    ) -> AsyncIterator[StreamPart]:
        """Stream results using SSE."""
        request_headers, content = await _aencode_json(json)
        request_headers["Accept"] = "text/event-stream"
        request_headers["Cache-Control"] = "no-store"
        # Add runtime headers with precedence
        if headers:
            request_headers.update(headers)

        reconnect_headers = {
            key: value
            for key, value in request_headers.items()
            if key.lower() not in {"content-length", "content-type"}
        }

        last_event_id: str | None = None
        reconnect_path: str | None = None
        reconnect_attempts = 0
        max_reconnect_attempts = 5

        while True:
            current_headers = dict(
                request_headers if reconnect_path is None else reconnect_headers
            )
            if last_event_id is not None:
                current_headers["Last-Event-ID"] = last_event_id

            current_method = method if reconnect_path is None else "GET"
            current_content = content if reconnect_path is None else None
            current_params = params if reconnect_path is None else None

            retry = False
            async with self.client.stream(
                current_method,
                reconnect_path or path,
                headers=current_headers,
                content=current_content,
                params=current_params,
            ) as res:
                if reconnect_path is None and on_response:
                    on_response(res)
                # check status
                await _araise_for_status_typed(res)
                # check content type
                content_type = res.headers.get("content-type", "").partition(";")[0]
                if "text/event-stream" not in content_type:
                    raise httpx.TransportError(
                        "Expected response header Content-Type to contain 'text/event-stream', "
                        f"got {content_type!r}"
                    )

                reconnect_location = res.headers.get("location")
                if reconnect_location:
                    reconnect_path = reconnect_location

                # parse SSE
                decoder = SSEDecoder()
                try:
                    async for line in aiter_lines_raw(res):
                        sse = decoder.decode(line=line.rstrip(b"\n"))
                        if sse is not None:
                            if decoder.last_event_id is not None:
                                last_event_id = decoder.last_event_id
                            if sse.event or sse.data is not None:
                                yield sse
                except httpx.HTTPError:
                    # httpx.TransportError inherits from HTTPError, so transient
                    # disconnects during streaming land here.
                    if reconnect_path is None:
                        raise
                    retry = True
                else:
                    if sse := decoder.decode(b""):
                        if decoder.last_event_id is not None:
                            last_event_id = decoder.last_event_id
                        if sse.event or sse.data is not None:
                            # decoder.decode(b"") flushes the in-flight event and may
                            # return an empty placeholder when there is no pending
                            # message. Skip these no-op events so the stream doesn't
                            # emit a trailing blank item after reconnects.
                            yield sse
            if retry:
                reconnect_attempts += 1
                if reconnect_attempts > max_reconnect_attempts:
                    raise httpx.TransportError(
                        "Exceeded maximum SSE reconnection attempts"
                    )
                continue
            break


async def _aencode_json(json: Any) -> tuple[dict[str, str], bytes | None]:
    if json is None:
        return {}, None
    body = await asyncio.get_running_loop().run_in_executor(
        None,
        orjson.dumps,
        json,
        _orjson_default,
        orjson.OPT_SERIALIZE_NUMPY | orjson.OPT_NON_STR_KEYS,
    )
    content_length = str(len(body))
    content_type = "application/json"
    headers = {"Content-Length": content_length, "Content-Type": content_type}
    return headers, body


async def _adecode_json(r: httpx.Response) -> Any:
    body = await r.aread()
    return (
        await asyncio.get_running_loop().run_in_executor(None, orjson.loads, body)
        if body
        else None
    )


class AssistantsClient:
    """Client for managing assistants in LangGraph.

    This class provides methods to interact with assistants,
    which are versioned configurations of your graph.

    ???+ example "Example"

        ```python
        client = get_client(url="http://localhost:2024")
        assistant = await client.assistants.get("assistant_id_123")
        ```
    """

    def __init__(self, http: HttpClient) -> None:
        self.http = http

    async def get(
        self,
        assistant_id: str,
        *,
        headers: Mapping[str, str] | None = None,
        params: QueryParamTypes | None = None,
    ) -> Assistant:
        """Get an assistant by ID.

        Args:
            assistant_id: The ID of the assistant to get.
            headers: Optional custom headers to include with the request.
            params: Optional query parameters to include with the request.

        Returns:
            Assistant: Assistant Object.

        ???+ example "Example Usage"

            ```python
            assistant = await client.assistants.get(
                assistant_id="my_assistant_id"
            )
            print(assistant)
            ```

            ```shell
            ----------------------------------------------------

            {
                'assistant_id': 'my_assistant_id',
                'graph_id': 'agent',
                'created_at': '2024-06-25T17:10:33.109781+00:00',
                'updated_at': '2024-06-25T17:10:33.109781+00:00',
                'config': {},
                'metadata': {'created_by': 'system'},
                'version': 1,
                'name': 'my_assistant'
            }
            ```
        """  # noqa: E501
        return await self.http.get(
            f"/assistants/{assistant_id}", headers=headers, params=params
        )

    async def get_graph(
        self,
        assistant_id: str,
        *,
        xray: int | bool = False,
        headers: Mapping[str, str] | None = None,
        params: QueryParamTypes | None = None,
    ) -> dict[str, list[dict[str, Any]]]:
        """Get the graph of an assistant by ID.

        Args:
            assistant_id: The ID of the assistant to get the graph of.
            xray: Include graph representation of subgraphs. If an integer value is provided, only subgraphs with a depth less than or equal to the value will be included.
            headers: Optional custom headers to include with the request.
            params: Optional query parameters to include with the request.

        Returns:
            Graph: The graph information for the assistant in JSON format.

        ???+ example "Example Usage"

            ```python
            client = get_client(url="http://localhost:2024")
            graph_info = await client.assistants.get_graph(
                assistant_id="my_assistant_id"
            )
            print(graph_info)
            ```

            ```shell

            --------------------------------------------------------------------------------------------------------------------------

            {
                'nodes':
                    [
                        {'id': '__start__', 'type': 'schema', 'data': '__start__'},
                        {'id': '__end__', 'type': 'schema', 'data': '__end__'},
                        {'id': 'agent','type': 'runnable','data': {'id': ['langgraph', 'utils', 'RunnableCallable'],'name': 'agent'}},
                    ],
                'edges':
                    [
                        {'source': '__start__', 'target': 'agent'},
                        {'source': 'agent','target': '__end__'}
                    ]
            }
            ```


        """  # noqa: E501
        query_params = {"xray": xray}
        if params:
            query_params.update(params)

        return await self.http.get(
            f"/assistants/{assistant_id}/graph", params=query_params, headers=headers
        )

    async def get_schemas(
        self,
        assistant_id: str,
        *,
        headers: Mapping[str, str] | None = None,
        params: QueryParamTypes | None = None,
    ) -> GraphSchema:
        """Get the schemas of an assistant by ID.

        Args:
            assistant_id: The ID of the assistant to get the schema of.
            headers: Optional custom headers to include with the request.
            params: Optional query parameters to include with the request.

        Returns:
            GraphSchema: The graph schema for the assistant.

        ???+ example "Example Usage"

            ```python
            client = get_client(url="http://localhost:2024")
            schema = await client.assistants.get_schemas(
                assistant_id="my_assistant_id"
            )
            print(schema)
            ```

            ```shell

            ----------------------------------------------------------------------------------------------------------------------------

            {
                'graph_id': 'agent',
                'state_schema':
                    {
                        'title': 'LangGraphInput',
                        '$ref': '#/definitions/AgentState',
                        'definitions':
                            {
                                'BaseMessage':
                                    {
                                        'title': 'BaseMessage',
                                        'description': 'Base abstract Message class. Messages are the inputs and outputs of ChatModels.',
                                        'type': 'object',
                                        'properties':
                                            {
                                             'content':
                                                {
                                                    'title': 'Content',
                                                    'anyOf': [
                                                        {'type': 'string'},
                                                        {'type': 'array','items': {'anyOf': [{'type': 'string'}, {'type': 'object'}]}}
                                                    ]
                                                },
                                            'additional_kwargs':
                                                {
                                                    'title': 'Additional Kwargs',
                                                    'type': 'object'
                                                },
                                            'response_metadata':
                                                {
                                                    'title': 'Response Metadata',
                                                    'type': 'object'
                                                },
                                            'type':
                                                {
                                                    'title': 'Type',
                                                    'type': 'string'
                                                },
                                            'name':
                                                {
                                                    'title': 'Name',
                                                    'type': 'string'
                                                },
                                            'id':
                                                {
                                                    'title': 'Id',
                                                    'type': 'string'
                                                }
                                            },
                                        'required': ['content', 'type']
                                    },
                                'AgentState':
                                    {
                                        'title': 'AgentState',
                                        'type': 'object',
                                        'properties':
                                            {
                                                'messages':
                                                    {
                                                        'title': 'Messages',
                                                        'type': 'array',
                                                        'items': {'$ref': '#/definitions/BaseMessage'}
                                                    }
                                            },
                                        'required': ['messages']
                                    }
                            }
                    },
                'context_schema':
                    {
                        'title': 'Context',
                        'type': 'object',
                        'properties':
                            {
                                'model_name':
                                    {
                                        'title': 'Model Name',
                                        'enum': ['anthropic', 'openai'],
                                        'type': 'string'
                                    }
                            }
                    }
            }
            ```

        """  # noqa: E501
        return await self.http.get(
            f"/assistants/{assistant_id}/schemas", headers=headers, params=params
        )

    async def get_subgraphs(
        self,
        assistant_id: str,
        namespace: str | None = None,
        recurse: bool = False,
        *,
        headers: Mapping[str, str] | None = None,
        params: QueryParamTypes | None = None,
    ) -> Subgraphs:
        """Get the schemas of an assistant by ID.

        Args:
            assistant_id: The ID of the assistant to get the schema of.
            namespace: Optional namespace to filter by.
            recurse: Whether to recursively get subgraphs.
            headers: Optional custom headers to include with the request.
            params: Optional query parameters to include with the request.

        Returns:
            Subgraphs: The graph schema for the assistant.

        """  # noqa: E501
        get_params = {"recurse": recurse}
        if params:
            get_params = {**get_params, **params}
        if namespace is not None:
            return await self.http.get(
                f"/assistants/{assistant_id}/subgraphs/{namespace}",
                params=get_params,
                headers=headers,
            )
        else:
            return await self.http.get(
                f"/assistants/{assistant_id}/subgraphs",
                params=get_params,
                headers=headers,
            )

    async def create(
        self,
        graph_id: str | None,
        config: Config | None = None,
        *,
        context: Context | None = None,
        metadata: Json = None,
        assistant_id: str | None = None,
        if_exists: OnConflictBehavior | None = None,
        name: str | None = None,
        headers: Mapping[str, str] | None = None,
        description: str | None = None,
        params: QueryParamTypes | None = None,
    ) -> Assistant:
        """Create a new assistant.

        Useful when graph is configurable and you want to create different assistants based on different configurations.

        Args:
            graph_id: The ID of the graph the assistant should use. The graph ID is normally set in your langgraph.json configuration.
            config: Configuration to use for the graph.
            metadata: Metadata to add to assistant.
            context: Static context to add to the assistant.
                !!! version-added "Added in version 0.6.0"
            assistant_id: Assistant ID to use, will default to a random UUID if not provided.
            if_exists: How to handle duplicate creation. Defaults to 'raise' under the hood.
                Must be either 'raise' (raise error if duplicate), or 'do_nothing' (return existing assistant).
            name: The name of the assistant. Defaults to 'Untitled' under the hood.
            headers: Optional custom headers to include with the request.
            description: Optional description of the assistant.
                The description field is available for langgraph-api server version>=0.0.45
            params: Optional query parameters to include with the request.

        Returns:
            Assistant: The created assistant.

        ???+ example "Example Usage"

            ```python
            client = get_client(url="http://localhost:2024")
            assistant = await client.assistants.create(
                graph_id="agent",
                context={"model_name": "openai"},
                metadata={"number":1},
                assistant_id="my-assistant-id",
                if_exists="do_nothing",
                name="my_name"
            )
            ```
        """  # noqa: E501
        payload: dict[str, Any] = {
            "graph_id": graph_id,
        }
        if config:
            payload["config"] = config
        if context:
            payload["context"] = context
        if metadata:
            payload["metadata"] = metadata
        if assistant_id:
            payload["assistant_id"] = assistant_id
        if if_exists:
            payload["if_exists"] = if_exists
        if name:
            payload["name"] = name
        if description:
            payload["description"] = description
        return await self.http.post(
            "/assistants", json=payload, headers=headers, params=params
        )

    async def update(
        self,
        assistant_id: str,
        *,
        graph_id: str | None = None,
        config: Config | None = None,
        context: Context | None = None,
        metadata: Json = None,
        name: str | None = None,
        headers: Mapping[str, str] | None = None,
        description: str | None = None,
        params: QueryParamTypes | None = None,
    ) -> Assistant:
        """Update an assistant.

        Use this to point to a different graph, update the configuration, or change the metadata of an assistant.

        Args:
            assistant_id: Assistant to update.
            graph_id: The ID of the graph the assistant should use.
                The graph ID is normally set in your langgraph.json configuration. If `None`, assistant will keep pointing to same graph.
            config: Configuration to use for the graph.
            context: Static context to add to the assistant.
                !!! version-added "Added in version 0.6.0"
            metadata: Metadata to merge with existing assistant metadata.
            name: The new name for the assistant.
            headers: Optional custom headers to include with the request.
            description: Optional description of the assistant.
                The description field is available for langgraph-api server version>=0.0.45
            params: Optional query parameters to include with the request.

        Returns:
            The updated assistant.

        ???+ example "Example Usage"

            ```python
            client = get_client(url="http://localhost:2024")
            assistant = await client.assistants.update(
                assistant_id='e280dad7-8618-443f-87f1-8e41841c180f',
                graph_id="other-graph",
                context={"model_name": "anthropic"},
                metadata={"number":2}
            )
            ```

        """  # noqa: E501
        payload: dict[str, Any] = {}
        if graph_id:
            payload["graph_id"] = graph_id
        if config:
            payload["config"] = config
        if context:
            payload["context"] = context
        if metadata:
            payload["metadata"] = metadata
        if name:
            payload["name"] = name
        if description:
            payload["description"] = description
        return await self.http.patch(
            f"/assistants/{assistant_id}",
            json=payload,
            headers=headers,
            params=params,
        )

    async def delete(
        self,
        assistant_id: str,
        *,
        headers: Mapping[str, str] | None = None,
        params: QueryParamTypes | None = None,
    ) -> None:
        """Delete an assistant.

        Args:
            assistant_id: The assistant ID to delete.
            headers: Optional custom headers to include with the request.
            params: Optional query parameters to include with the request.

        Returns:
            `None`

        ???+ example "Example Usage"

            ```python
            client = get_client(url="http://localhost:2024")
            await client.assistants.delete(
                assistant_id="my_assistant_id"
            )
            ```

        """  # noqa: E501
        await self.http.delete(
            f"/assistants/{assistant_id}", headers=headers, params=params
        )

    async def search(
        self,
        *,
        metadata: Json = None,
        graph_id: str | None = None,
        limit: int = 10,
        offset: int = 0,
        sort_by: AssistantSortBy | None = None,
        sort_order: SortOrder | None = None,
        select: list[AssistantSelectField] | None = None,
        headers: Mapping[str, str] | None = None,
        params: QueryParamTypes | None = None,
    ) -> list[Assistant]:
        """Search for assistants.

        Args:
            metadata: Metadata to filter by. Exact match filter for each KV pair.
            graph_id: The ID of the graph to filter by.
                The graph ID is normally set in your langgraph.json configuration.
            limit: The maximum number of results to return.
            offset: The number of results to skip.
            sort_by: The field to sort by.
            sort_order: The order to sort by.
            headers: Optional custom headers to include with the request.
            params: Optional query parameters to include with the request.

        Returns:
            A list of assistants.

        ???+ example "Example Usage"

            ```python
            client = get_client(url="http://localhost:2024")
            assistants = await client.assistants.search(
                metadata = {"name":"my_name"},
                graph_id="my_graph_id",
                limit=5,
                offset=5
            )
            ```
        """
        payload: dict[str, Any] = {
            "limit": limit,
            "offset": offset,
        }
        if metadata:
            payload["metadata"] = metadata
        if graph_id:
            payload["graph_id"] = graph_id
        if sort_by:
            payload["sort_by"] = sort_by
        if sort_order:
            payload["sort_order"] = sort_order
        if select:
            payload["select"] = select
        return await self.http.post(
            "/assistants/search",
            json=payload,
            headers=headers,
            params=params,
        )

    async def count(
        self,
        *,
        metadata: Json = None,
        graph_id: str | None = None,
        headers: Mapping[str, str] | None = None,
        params: QueryParamTypes | None = None,
    ) -> int:
        """Count assistants matching filters.

        Args:
            metadata: Metadata to filter by. Exact match for each key/value.
            graph_id: Optional graph id to filter by.
            headers: Optional custom headers to include with the request.
            params: Optional query parameters to include with the request.

        Returns:
            int: Number of assistants matching the criteria.
        """
        payload: dict[str, Any] = {}
        if metadata:
            payload["metadata"] = metadata
        if graph_id:
            payload["graph_id"] = graph_id
        return await self.http.post(
            "/assistants/count", json=payload, headers=headers, params=params
        )

    async def get_versions(
        self,
        assistant_id: str,
        metadata: Json = None,
        limit: int = 10,
        offset: int = 0,
        *,
        headers: Mapping[str, str] | None = None,
        params: QueryParamTypes | None = None,
    ) -> list[AssistantVersion]:
        """List all versions of an assistant.

        Args:
            assistant_id: The assistant ID to get versions for.
            metadata: Metadata to filter versions by. Exact match filter for each KV pair.
            limit: The maximum number of versions to return.
            offset: The number of versions to skip.
            headers: Optional custom headers to include with the request.
            params: Optional query parameters to include with the request.

        Returns:
            A list of assistant versions.

        ???+ example "Example Usage"

            ```python
            client = get_client(url="http://localhost:2024")
            assistant_versions = await client.assistants.get_versions(
                assistant_id="my_assistant_id"
            )
            ```
        """  # noqa: E501

        payload: dict[str, Any] = {
            "limit": limit,
            "offset": offset,
        }
        if metadata:
            payload["metadata"] = metadata
        return await self.http.post(
            f"/assistants/{assistant_id}/versions",
            json=payload,
            headers=headers,
            params=params,
        )

    async def set_latest(
        self,
        assistant_id: str,
        version: int,
        *,
        headers: Mapping[str, str] | None = None,
        params: QueryParamTypes | None = None,
    ) -> Assistant:
        """Change the version of an assistant.

        Args:
            assistant_id: The assistant ID to delete.
            version: The version to change to.
            headers: Optional custom headers to include with the request.
            params: Optional query parameters to include with the request.

        Returns:
            Assistant Object.

        ???+ example "Example Usage"

            ```python
            client = get_client(url="http://localhost:2024")
            new_version_assistant = await client.assistants.set_latest(
                assistant_id="my_assistant_id",
                version=3
            )
            ```

        """  # noqa: E501

        payload: dict[str, Any] = {"version": version}

        return await self.http.post(
            f"/assistants/{assistant_id}/latest",
            json=payload,
            headers=headers,
            params=params,
        )


class ThreadsClient:
    """Client for managing threads in LangGraph.

    A thread maintains the state of a graph across multiple interactions/invocations (aka runs).
    It accumulates and persists the graph's state, allowing for continuity between separate
    invocations of the graph.

    ???+ example "Example"

        ```python
        client = get_client(url="http://localhost:2024"))
        new_thread = await client.threads.create(metadata={"user_id": "123"})
        ```
    """

    def __init__(self, http: HttpClient) -> None:
        self.http = http

    async def get(
        self,
        thread_id: str,
        *,
        headers: Mapping[str, str] | None = None,
        params: QueryParamTypes | None = None,
    ) -> Thread:
        """Get a thread by ID.

        Args:
            thread_id: The ID of the thread to get.
            headers: Optional custom headers to include with the request.
            params: Optional query parameters to include with the request.

        Returns:
            Thread object.

        ???+ example "Example Usage"

            ```python
            client = get_client(url="http://localhost:2024")
            thread = await client.threads.get(
                thread_id="my_thread_id"
            )
            print(thread)
            ```

            ```shell
            -----------------------------------------------------

            {
                'thread_id': 'my_thread_id',
                'created_at': '2024-07-18T18:35:15.540834+00:00',
                'updated_at': '2024-07-18T18:35:15.540834+00:00',
                'metadata': {'graph_id': 'agent'}
            }
            ```

        """  # noqa: E501

        return await self.http.get(
            f"/threads/{thread_id}", headers=headers, params=params
        )

    async def create(
        self,
        *,
        metadata: Json = None,
        thread_id: str | None = None,
        if_exists: OnConflictBehavior | None = None,
        supersteps: Sequence[dict[str, Sequence[dict[str, Any]]]] | None = None,
        graph_id: str | None = None,
        ttl: int | Mapping[str, Any] | None = None,
        headers: Mapping[str, str] | None = None,
        params: QueryParamTypes | None = None,
    ) -> Thread:
        """Create a new thread.

        Args:
            metadata: Metadata to add to thread.
            thread_id: ID of thread.
                If `None`, ID will be a randomly generated UUID.
            if_exists: How to handle duplicate creation. Defaults to 'raise' under the hood.
                Must be either 'raise' (raise error if duplicate), or 'do_nothing' (return existing thread).
            supersteps: Apply a list of supersteps when creating a thread, each containing a sequence of updates.
                Each update has `values` or `command` and `as_node`. Used for copying a thread between deployments.
            graph_id: Optional graph ID to associate with the thread.
            ttl: Optional time-to-live in minutes for the thread. You can pass an
                integer (minutes) or a mapping with keys `ttl` and optional
                `strategy` (defaults to "delete").
            headers: Optional custom headers to include with the request.
            params: Optional query parameters to include with the request.

        Returns:
            The created thread.

        ???+ example "Example Usage"

            ```python
            client = get_client(url="http://localhost:2024")
            thread = await client.threads.create(
                metadata={"number":1},
                thread_id="my-thread-id",
                if_exists="raise"
            )
            ```
        """  # noqa: E501
        payload: dict[str, Any] = {}
        if thread_id:
            payload["thread_id"] = thread_id
        if metadata or graph_id:
            payload["metadata"] = {
                **(metadata or {}),
                **({"graph_id": graph_id} if graph_id else {}),
            }
        if if_exists:
            payload["if_exists"] = if_exists
        if supersteps:
            payload["supersteps"] = [
                {
                    "updates": [
                        {
                            "values": u["values"],
                            "command": u.get("command"),
                            "as_node": u["as_node"],
                        }
                        for u in s["updates"]
                    ]
                }
                for s in supersteps
            ]
        if ttl is not None:
            if isinstance(ttl, (int, float)):
                payload["ttl"] = {"ttl": ttl, "strategy": "delete"}
            else:
                payload["ttl"] = ttl

        return await self.http.post(
            "/threads", json=payload, headers=headers, params=params
        )

    async def update(
        self,
        thread_id: str,
        *,
        metadata: Mapping[str, Any],
        ttl: int | Mapping[str, Any] | None = None,
        headers: Mapping[str, str] | None = None,
        params: QueryParamTypes | None = None,
    ) -> Thread:
        """Update a thread.

        Args:
            thread_id: ID of thread to update.
            metadata: Metadata to merge with existing thread metadata.
            ttl: Optional time-to-live in minutes for the thread. You can pass an
                integer (minutes) or a mapping with keys `ttl` and optional
                `strategy` (defaults to "delete").
            headers: Optional custom headers to include with the request.
            params: Optional query parameters to include with the request.

        Returns:
            The created thread.

        ???+ example "Example Usage"

            ```python
            client = get_client(url="http://localhost:2024")
            thread = await client.threads.update(
                thread_id="my-thread-id",
                metadata={"number":1},
                ttl=43_200,
            )
            ```
        """  # noqa: E501
        payload: dict[str, Any] = {"metadata": metadata}
        if ttl is not None:
            if isinstance(ttl, (int, float)):
                payload["ttl"] = {"ttl": ttl, "strategy": "delete"}
            else:
                payload["ttl"] = ttl
        return await self.http.patch(
            f"/threads/{thread_id}",
            json=payload,
            headers=headers,
            params=params,
        )

    async def delete(
        self,
        thread_id: str,
        *,
        headers: Mapping[str, str] | None = None,
        params: QueryParamTypes | None = None,
    ) -> None:
        """Delete a thread.

        Args:
            thread_id: The ID of the thread to delete.
            headers: Optional custom headers to include with the request.
            params: Optional query parameters to include with the request.

        Returns:
            `None`

        ???+ example "Example Usage"

            ```python
            client = get_client(url="http://localhost2024)
            await client.threads.delete(
                thread_id="my_thread_id"
            )
            ```

        """  # noqa: E501
        await self.http.delete(f"/threads/{thread_id}", headers=headers, params=params)

    async def search(
        self,
        *,
        metadata: Json = None,
        values: Json = None,
        ids: Sequence[str] | None = None,
        status: ThreadStatus | None = None,
        limit: int = 10,
        offset: int = 0,
        sort_by: ThreadSortBy | None = None,
        sort_order: SortOrder | None = None,
        select: list[ThreadSelectField] | None = None,
        headers: Mapping[str, str] | None = None,
        params: QueryParamTypes | None = None,
    ) -> list[Thread]:
        """Search for threads.

        Args:
            metadata: Thread metadata to filter on.
            values: State values to filter on.
            ids: List of thread IDs to filter by.
            status: Thread status to filter on.
                Must be one of 'idle', 'busy', 'interrupted' or 'error'.
            limit: Limit on number of threads to return.
            offset: Offset in threads table to start search from.
            sort_by: Sort by field.
            sort_order: Sort order.
            headers: Optional custom headers to include with the request.
            params: Optional query parameters to include with the request.

        Returns:
            List of the threads matching the search parameters.

        ???+ example "Example Usage"

            ```python
            client = get_client(url="http://localhost:2024")
            threads = await client.threads.search(
                metadata={"number":1},
                status="interrupted",
                limit=15,
                offset=5
            )
            ```

        """  # noqa: E501
        payload: dict[str, Any] = {
            "limit": limit,
            "offset": offset,
        }
        if metadata:
            payload["metadata"] = metadata
        if values:
            payload["values"] = values
        if ids:
            payload["ids"] = ids
        if status:
            payload["status"] = status
        if sort_by:
            payload["sort_by"] = sort_by
        if sort_order:
            payload["sort_order"] = sort_order
        if select:
            payload["select"] = select
        return await self.http.post(
            "/threads/search",
            json=payload,
            headers=headers,
            params=params,
        )

    async def count(
        self,
        *,
        metadata: Json = None,
        values: Json = None,
        status: ThreadStatus | None = None,
        headers: Mapping[str, str] | None = None,
        params: QueryParamTypes | None = None,
    ) -> int:
        """Count threads matching filters.

        Args:
            metadata: Thread metadata to filter on.
            values: State values to filter on.
            status: Thread status to filter on.
            headers: Optional custom headers to include with the request.
            params: Optional query parameters to include with the request.

        Returns:
            int: Number of threads matching the criteria.
        """
        payload: dict[str, Any] = {}
        if metadata:
            payload["metadata"] = metadata
        if values:
            payload["values"] = values
        if status:
            payload["status"] = status
        return await self.http.post(
            "/threads/count", json=payload, headers=headers, params=params
        )

    async def copy(
        self,
        thread_id: str,
        *,
        headers: Mapping[str, str] | None = None,
        params: QueryParamTypes | None = None,
    ) -> None:
        """Copy a thread.

        Args:
            thread_id: The ID of the thread to copy.
            headers: Optional custom headers to include with the request.
            params: Optional query parameters to include with the request.

        Returns:
            `None`

        ???+ example "Example Usage"

            ```python
            client = get_client(url="http://localhost:2024)
            await client.threads.copy(
                thread_id="my_thread_id"
            )
            ```

        """  # noqa: E501
        return await self.http.post(
            f"/threads/{thread_id}/copy", json=None, headers=headers, params=params
        )

    async def get_state(
        self,
        thread_id: str,
        checkpoint: Checkpoint | None = None,
        checkpoint_id: str | None = None,  # deprecated
        *,
        subgraphs: bool = False,
        headers: Mapping[str, str] | None = None,
        params: QueryParamTypes | None = None,
    ) -> ThreadState:
        """Get the state of a thread.

        Args:
            thread_id: The ID of the thread to get the state of.
            checkpoint: The checkpoint to get the state of.
            checkpoint_id: (deprecated) The checkpoint ID to get the state of.
            subgraphs: Include subgraphs states.
            headers: Optional custom headers to include with the request.
            params: Optional query parameters to include with the request.

        Returns:
            The thread of the state.

        ???+ example "Example Usage"

            ```python
            client = get_client(url="http://localhost:2024)
            thread_state = await client.threads.get_state(
                thread_id="my_thread_id",
                checkpoint_id="my_checkpoint_id"
            )
            print(thread_state)
            ```

            ```shell
            ----------------------------------------------------------------------------------------------------------------------------------------------------------------------

            {
                'values': {
                    'messages': [
                        {
                            'content': 'how are you?',
                            'additional_kwargs': {},
                            'response_metadata': {},
                            'type': 'human',
                            'name': None,
                            'id': 'fe0a5778-cfe9-42ee-b807-0adaa1873c10',
                            'example': False
                        },
                        {
                            'content': "I'm doing well, thanks for asking! I'm an AI assistant created by Anthropic to be helpful, honest, and harmless.",
                            'additional_kwargs': {},
                            'response_metadata': {},
                            'type': 'ai',
                            'name': None,
                            'id': 'run-159b782c-b679-4830-83c6-cef87798fe8b',
                            'example': False,
                            'tool_calls': [],
                            'invalid_tool_calls': [],
                            'usage_metadata': None
                        }
                    ]
                },
                'next': [],
                'checkpoint':
                    {
                        'thread_id': 'e2496803-ecd5-4e0c-a779-3226296181c2',
                        'checkpoint_ns': '',
                        'checkpoint_id': '1ef4a9b8-e6fb-67b1-8001-abd5184439d1'
                    }
                'metadata':
                    {
                        'step': 1,
                        'run_id': '1ef4a9b8-d7da-679a-a45a-872054341df2',
                        'source': 'loop',
                        'writes':
                            {
                                'agent':
                                    {
                                        'messages': [
                                            {
                                                'id': 'run-159b782c-b679-4830-83c6-cef87798fe8b',
                                                'name': None,
                                                'type': 'ai',
                                                'content': "I'm doing well, thanks for asking! I'm an AI assistant created by Anthropic to be helpful, honest, and harmless.",
                                                'example': False,
                                                'tool_calls': [],
                                                'usage_metadata': None,
                                                'additional_kwargs': {},
                                                'response_metadata': {},
                                                'invalid_tool_calls': []
                                            }
                                        ]
                                    }
                            },
                'user_id': None,
                'graph_id': 'agent',
                'thread_id': 'e2496803-ecd5-4e0c-a779-3226296181c2',
                'created_by': 'system',
                'assistant_id': 'fe096781-5601-53d2-b2f6-0d3403f7e9ca'},
                'created_at': '2024-07-25T15:35:44.184703+00:00',
                'parent_config':
                    {
                        'thread_id': 'e2496803-ecd5-4e0c-a779-3226296181c2',
                        'checkpoint_ns': '',
                        'checkpoint_id': '1ef4a9b8-d80d-6fa7-8000-9300467fad0f'
                    }
            }
            ```
        """  # noqa: E501
        if checkpoint:
            return await self.http.post(
                f"/threads/{thread_id}/state/checkpoint",
                json={"checkpoint": checkpoint, "subgraphs": subgraphs},
                headers=headers,
                params=params,
            )
        elif checkpoint_id:
            get_params = {"subgraphs": subgraphs}
            if params:
                get_params = {**get_params, **params}
            return await self.http.get(
                f"/threads/{thread_id}/state/{checkpoint_id}",
                params=get_params,
                headers=headers,
            )
        else:
            get_params = {"subgraphs": subgraphs}
            if params:
                get_params = {**get_params, **params}
            return await self.http.get(
                f"/threads/{thread_id}/state",
                params=get_params,
                headers=headers,
            )

    async def update_state(
        self,
        thread_id: str,
        values: dict[str, Any] | Sequence[dict] | None,
        *,
        as_node: str | None = None,
        checkpoint: Checkpoint | None = None,
        checkpoint_id: str | None = None,  # deprecated
        headers: Mapping[str, str] | None = None,
        params: QueryParamTypes | None = None,
    ) -> ThreadUpdateStateResponse:
        """Update the state of a thread.

        Args:
            thread_id: The ID of the thread to update.
            values: The values to update the state with.
            as_node: Update the state as if this node had just executed.
            checkpoint: The checkpoint to update the state of.
            checkpoint_id: (deprecated) The checkpoint ID to update the state of.
            headers: Optional custom headers to include with the request.
            params: Optional query parameters to include with the request.

        Returns:
            Response after updating a thread's state.

        ???+ example "Example Usage"

            ```python
            client = get_client(url="http://localhost:2024)
            response = await client.threads.update_state(
                thread_id="my_thread_id",
                values={"messages":[{"role": "user", "content": "hello!"}]},
                as_node="my_node",
            )
            print(response)
            ```
            ```shell

            ----------------------------------------------------------------------------------------------------------------------------------------------------------------------

            {
                'checkpoint': {
                    'thread_id': 'e2496803-ecd5-4e0c-a779-3226296181c2',
                    'checkpoint_ns': '',
                    'checkpoint_id': '1ef4a9b8-e6fb-67b1-8001-abd5184439d1',
                    'checkpoint_map': {}
                }
            }
            ```
        """  # noqa: E501
        payload: dict[str, Any] = {
            "values": values,
        }
        if checkpoint_id:
            payload["checkpoint_id"] = checkpoint_id
        if checkpoint:
            payload["checkpoint"] = checkpoint
        if as_node:
            payload["as_node"] = as_node
        return await self.http.post(
            f"/threads/{thread_id}/state", json=payload, headers=headers, params=params
        )

    async def get_history(
        self,
        thread_id: str,
        *,
        limit: int = 10,
        before: str | Checkpoint | None = None,
        metadata: Mapping[str, Any] | None = None,
        checkpoint: Checkpoint | None = None,
        headers: Mapping[str, str] | None = None,
        params: QueryParamTypes | None = None,
    ) -> list[ThreadState]:
        """Get the state history of a thread.

        Args:
            thread_id: The ID of the thread to get the state history for.
            checkpoint: Return states for this subgraph. If empty defaults to root.
            limit: The maximum number of states to return.
            before: Return states before this checkpoint.
            metadata: Filter states by metadata key-value pairs.
            headers: Optional custom headers to include with the request.
            params: Optional query parameters to include with the request.

        Returns:
            The state history of the thread.

        ???+ example "Example Usage"

            ```python
            client = get_client(url="http://localhost:2024)
            thread_state = await client.threads.get_history(
                thread_id="my_thread_id",
                limit=5,
            )
            ```

        """  # noqa: E501
        payload: dict[str, Any] = {
            "limit": limit,
        }
        if before:
            payload["before"] = before
        if metadata:
            payload["metadata"] = metadata
        if checkpoint:
            payload["checkpoint"] = checkpoint
        return await self.http.post(
            f"/threads/{thread_id}/history",
            json=payload,
            headers=headers,
            params=params,
        )

    async def join_stream(
        self,
        thread_id: str,
        *,
        last_event_id: str | None = None,
        stream_mode: ThreadStreamMode | Sequence[ThreadStreamMode] = "run_modes",
        headers: Mapping[str, str] | None = None,
        params: QueryParamTypes | None = None,
    ) -> AsyncIterator[StreamPart]:
        """Get a stream of events for a thread.

        Args:
            thread_id: The ID of the thread to get the stream for.
            last_event_id: The ID of the last event to get.
            headers: Optional custom headers to include with the request.
            params: Optional query parameters to include with the request.

        Returns:
            An iterator of stream parts.

        ???+ example "Example Usage"

            ```python

            for chunk in client.threads.join_stream(
                thread_id="my_thread_id",
                last_event_id="my_event_id",
            ):
                print(chunk)
            ```

        """  # noqa: E501
        query_params = {
            "stream_mode": stream_mode,
        }
        if params:
            query_params.update(params)
        return self.http.stream(
            f"/threads/{thread_id}/stream",
            "GET",
            headers={
                **({"Last-Event-ID": last_event_id} if last_event_id else {}),
                **(headers or {}),
            },
            params=query_params,
        )


class RunsClient:
    """Client for managing runs in LangGraph.

    A run is a single assistant invocation with optional input, config, context, and metadata.
    This client manages runs, which can be stateful (on threads) or stateless.

    ???+ example "Example"

        ```python
        client = get_client(url="http://localhost:2024")
        run = await client.runs.create(assistant_id="asst_123", thread_id="thread_456", input={"query": "Hello"})
        ```
    """

    def __init__(self, http: HttpClient) -> None:
        self.http = http

    @overload
    def stream(
        self,
        thread_id: str,
        assistant_id: str,
        *,
        input: Mapping[str, Any] | None = None,
        command: Command | None = None,
        stream_mode: StreamMode | Sequence[StreamMode] = "values",
        stream_subgraphs: bool = False,
        stream_resumable: bool = False,
        metadata: Mapping[str, Any] | None = None,
        config: Config | None = None,
        context: Context | None = None,
        checkpoint: Checkpoint | None = None,
        checkpoint_id: str | None = None,
        checkpoint_during: bool | None = None,
        interrupt_before: All | Sequence[str] | None = None,
        interrupt_after: All | Sequence[str] | None = None,
        feedback_keys: Sequence[str] | None = None,
        on_disconnect: DisconnectMode | None = None,
        webhook: str | None = None,
        multitask_strategy: MultitaskStrategy | None = None,
        if_not_exists: IfNotExists | None = None,
        after_seconds: int | None = None,
        headers: Mapping[str, str] | None = None,
        params: QueryParamTypes | None = None,
        on_run_created: Callable[[RunCreateMetadata], None] | None = None,
    ) -> AsyncIterator[StreamPart]: ...

    @overload
    def stream(
        self,
        thread_id: None,
        assistant_id: str,
        *,
        input: Mapping[str, Any] | None = None,
        command: Command | None = None,
        stream_mode: StreamMode | Sequence[StreamMode] = "values",
        stream_subgraphs: bool = False,
        stream_resumable: bool = False,
        metadata: Mapping[str, Any] | None = None,
        config: Config | None = None,
        checkpoint_during: bool | None = None,
        interrupt_before: All | Sequence[str] | None = None,
        interrupt_after: All | Sequence[str] | None = None,
        feedback_keys: Sequence[str] | None = None,
        on_disconnect: DisconnectMode | None = None,
        on_completion: OnCompletionBehavior | None = None,
        if_not_exists: IfNotExists | None = None,
        webhook: str | None = None,
        after_seconds: int | None = None,
        headers: Mapping[str, str] | None = None,
        params: QueryParamTypes | None = None,
        on_run_created: Callable[[RunCreateMetadata], None] | None = None,
    ) -> AsyncIterator[StreamPart]: ...

    def stream(
        self,
        thread_id: str | None,
        assistant_id: str,
        *,
        input: Mapping[str, Any] | None = None,
        command: Command | None = None,
        stream_mode: StreamMode | Sequence[StreamMode] = "values",
        stream_subgraphs: bool = False,
        stream_resumable: bool = False,
        metadata: Mapping[str, Any] | None = None,
        config: Config | None = None,
        context: Context | None = None,
        checkpoint: Checkpoint | None = None,
        checkpoint_id: str | None = None,
        checkpoint_during: bool | None = None,  # deprecated
        interrupt_before: All | Sequence[str] | None = None,
        interrupt_after: All | Sequence[str] | None = None,
        feedback_keys: Sequence[str] | None = None,
        on_disconnect: DisconnectMode | None = None,
        on_completion: OnCompletionBehavior | None = None,
        webhook: str | None = None,
        multitask_strategy: MultitaskStrategy | None = None,
        if_not_exists: IfNotExists | None = None,
        after_seconds: int | None = None,
        headers: Mapping[str, str] | None = None,
        params: QueryParamTypes | None = None,
        on_run_created: Callable[[RunCreateMetadata], None] | None = None,
        durability: Durability | None = None,
    ) -> AsyncIterator[StreamPart]:
        """Create a run and stream the results.

        Args:
            thread_id: the thread ID to assign to the thread.
                If `None` will create a stateless run.
            assistant_id: The assistant ID or graph name to stream from.
                If using graph name, will default to first assistant created from that graph.
            input: The input to the graph.
            command: A command to execute. Cannot be combined with input.
            stream_mode: The stream mode(s) to use.
            stream_subgraphs: Whether to stream output from subgraphs.
            stream_resumable: Whether the stream is considered resumable.
                If true, the stream can be resumed and replayed in its entirety even after disconnection.
            metadata: Metadata to assign to the run.
            config: The configuration for the assistant.
            context: Static context to add to the assistant.
                !!! version-added "Added in version 0.6.0"
            checkpoint: The checkpoint to resume from.
            checkpoint_during: (deprecated) Whether to checkpoint during the run (or only at the end/interruption).
            interrupt_before: Nodes to interrupt immediately before they get executed.
            interrupt_after: Nodes to Nodes to interrupt immediately after they get executed.
            feedback_keys: Feedback keys to assign to run.
            on_disconnect: The disconnect mode to use.
                Must be one of 'cancel' or 'continue'.
            on_completion: Whether to delete or keep the thread created for a stateless run.
                Must be one of 'delete' or 'keep'.
            webhook: Webhook to call after LangGraph API call is done.
            multitask_strategy: Multitask strategy to use.
                Must be one of 'reject', 'interrupt', 'rollback', or 'enqueue'.
            if_not_exists: How to handle missing thread. Defaults to 'reject'.
                Must be either 'reject' (raise error if missing), or 'create' (create new thread).
            after_seconds: The number of seconds to wait before starting the run.
                Use to schedule future runs.
            headers: Optional custom headers to include with the request.
            params: Optional query parameters to include with the request.
            on_run_created: Callback when a run is created.
            durability: The durability to use for the run. Values are "sync", "async", or "exit".
                "async" means checkpoints are persisted async while next graph step executes, replaces checkpoint_during=True
                "sync" means checkpoints are persisted sync after graph step executes, replaces checkpoint_during=False
                "exit" means checkpoints are only persisted when the run exits, does not save intermediate steps

        Returns:
            Asynchronous iterator of stream results.

        ???+ example "Example Usage"

            ```python
            client = get_client(url="http://localhost:2024)
            async for chunk in client.runs.stream(
                thread_id=None,
                assistant_id="agent",
                input={"messages": [{"role": "user", "content": "how are you?"}]},
                stream_mode=["values","debug"],
                metadata={"name":"my_run"},
                context={"model_name": "anthropic"},
                interrupt_before=["node_to_stop_before_1","node_to_stop_before_2"],
                interrupt_after=["node_to_stop_after_1","node_to_stop_after_2"],
                feedback_keys=["my_feedback_key_1","my_feedback_key_2"],
                webhook="https://my.fake.webhook.com",
                multitask_strategy="interrupt"
            ):
                print(chunk)
            ```

            ```shell

            ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

            StreamPart(event='metadata', data={'run_id': '1ef4a9b8-d7da-679a-a45a-872054341df2'})
            StreamPart(event='values', data={'messages': [{'content': 'how are you?', 'additional_kwargs': {}, 'response_metadata': {}, 'type': 'human', 'name': None, 'id': 'fe0a5778-cfe9-42ee-b807-0adaa1873c10', 'example': False}]})
            StreamPart(event='values', data={'messages': [{'content': 'how are you?', 'additional_kwargs': {}, 'response_metadata': {}, 'type': 'human', 'name': None, 'id': 'fe0a5778-cfe9-42ee-b807-0adaa1873c10', 'example': False}, {'content': "I'm doing well, thanks for asking! I'm an AI assistant created by Anthropic to be helpful, honest, and harmless.", 'additional_kwargs': {}, 'response_metadata': {}, 'type': 'ai', 'name': None, 'id': 'run-159b782c-b679-4830-83c6-cef87798fe8b', 'example': False, 'tool_calls': [], 'invalid_tool_calls': [], 'usage_metadata': None}]})
            StreamPart(event='end', data=None)
            ```

        """  # noqa: E501
        if checkpoint_during is not None:
            warnings.warn(
                "`checkpoint_during` is deprecated and will be removed in a future version. Use `durability` instead.",
                DeprecationWarning,
                stacklevel=2,
            )

        payload = {
            "input": input,
            "command": (
                {k: v for k, v in command.items() if v is not None} if command else None
            ),
            "config": config,
            "context": context,
            "metadata": metadata,
            "stream_mode": stream_mode,
            "stream_subgraphs": stream_subgraphs,
            "stream_resumable": stream_resumable,
            "assistant_id": assistant_id,
            "interrupt_before": interrupt_before,
            "interrupt_after": interrupt_after,
            "feedback_keys": feedback_keys,
            "webhook": webhook,
            "checkpoint": checkpoint,
            "checkpoint_id": checkpoint_id,
            "checkpoint_during": checkpoint_during,
            "multitask_strategy": multitask_strategy,
            "if_not_exists": if_not_exists,
            "on_disconnect": on_disconnect,
            "on_completion": on_completion,
            "after_seconds": after_seconds,
            "durability": durability,
        }
        endpoint = (
            f"/threads/{thread_id}/runs/stream"
            if thread_id is not None
            else "/runs/stream"
        )

        def on_response(res: httpx.Response):
            """Callback function to handle the response."""
            if on_run_created and (metadata := _get_run_metadata_from_response(res)):
                on_run_created(metadata)

        return self.http.stream(
            endpoint,
            "POST",
            json={k: v for k, v in payload.items() if v is not None},
            params=params,
            headers=headers,
            on_response=on_response if on_run_created else None,
        )

    @overload
    async def create(
        self,
        thread_id: None,
        assistant_id: str,
        *,
        input: Mapping[str, Any] | None = None,
        command: Command | None = None,
        stream_mode: StreamMode | Sequence[StreamMode] = "values",
        stream_subgraphs: bool = False,
        stream_resumable: bool = False,
        metadata: Mapping[str, Any] | None = None,
        checkpoint_during: bool | None = None,
        config: Config | None = None,
        context: Context | None = None,
        interrupt_before: All | Sequence[str] | None = None,
        interrupt_after: All | Sequence[str] | None = None,
        webhook: str | None = None,
        on_completion: OnCompletionBehavior | None = None,
        if_not_exists: IfNotExists | None = None,
        after_seconds: int | None = None,
        headers: Mapping[str, str] | None = None,
        params: QueryParamTypes | None = None,
        on_run_created: Callable[[RunCreateMetadata], None] | None = None,
    ) -> Run: ...

    @overload
    async def create(
        self,
        thread_id: str,
        assistant_id: str,
        *,
        input: Mapping[str, Any] | None = None,
        command: Command | None = None,
        stream_mode: StreamMode | Sequence[StreamMode] = "values",
        stream_subgraphs: bool = False,
        stream_resumable: bool = False,
        metadata: Mapping[str, Any] | None = None,
        config: Config | None = None,
        context: Context | None = None,
        checkpoint: Checkpoint | None = None,
        checkpoint_id: str | None = None,
        checkpoint_during: bool | None = None,
        interrupt_before: All | Sequence[str] | None = None,
        interrupt_after: All | Sequence[str] | None = None,
        webhook: str | None = None,
        multitask_strategy: MultitaskStrategy | None = None,
        if_not_exists: IfNotExists | None = None,
        after_seconds: int | None = None,
        headers: Mapping[str, str] | None = None,
        params: QueryParamTypes | None = None,
        on_run_created: Callable[[RunCreateMetadata], None] | None = None,
    ) -> Run: ...

    async def create(
        self,
        thread_id: str | None,
        assistant_id: str,
        *,
        input: Mapping[str, Any] | None = None,
        command: Command | None = None,
        stream_mode: StreamMode | Sequence[StreamMode] = "values",
        stream_subgraphs: bool = False,
        stream_resumable: bool = False,
        metadata: Mapping[str, Any] | None = None,
        config: Config | None = None,
        context: Context | None = None,
        checkpoint: Checkpoint | None = None,
        checkpoint_id: str | None = None,
        checkpoint_during: bool | None = None,  # deprecated
        interrupt_before: All | Sequence[str] | None = None,
        interrupt_after: All | Sequence[str] | None = None,
        webhook: str | None = None,
        multitask_strategy: MultitaskStrategy | None = None,
        if_not_exists: IfNotExists | None = None,
        on_completion: OnCompletionBehavior | None = None,
        after_seconds: int | None = None,
        headers: Mapping[str, str] | None = None,
        params: QueryParamTypes | None = None,
        on_run_created: Callable[[RunCreateMetadata], None] | None = None,
        durability: Durability | None = None,
    ) -> Run:
        """Create a background run.

        Args:
            thread_id: the thread ID to assign to the thread.
                If `None` will create a stateless run.
            assistant_id: The assistant ID or graph name to stream from.
                If using graph name, will default to first assistant created from that graph.
            input: The input to the graph.
            command: A command to execute. Cannot be combined with input.
            stream_mode: The stream mode(s) to use.
            stream_subgraphs: Whether to stream output from subgraphs.
            stream_resumable: Whether the stream is considered resumable.
                If true, the stream can be resumed and replayed in its entirety even after disconnection.
            metadata: Metadata to assign to the run.
            config: The configuration for the assistant.
            context: Static context to add to the assistant.
                !!! version-added "Added in version 0.6.0"
            checkpoint: The checkpoint to resume from.
            checkpoint_during: (deprecated) Whether to checkpoint during the run (or only at the end/interruption).
            interrupt_before: Nodes to interrupt immediately before they get executed.
            interrupt_after: Nodes to Nodes to interrupt immediately after they get executed.
            webhook: Webhook to call after LangGraph API call is done.
            multitask_strategy: Multitask strategy to use.
                Must be one of 'reject', 'interrupt', 'rollback', or 'enqueue'.
            on_completion: Whether to delete or keep the thread created for a stateless run.
                Must be one of 'delete' or 'keep'.
            if_not_exists: How to handle missing thread. Defaults to 'reject'.
                Must be either 'reject' (raise error if missing), or 'create' (create new thread).
            after_seconds: The number of seconds to wait before starting the run.
                Use to schedule future runs.
            headers: Optional custom headers to include with the request.
            on_run_created: Optional callback to call when a run is created.
            durability: The durability to use for the run. Values are "sync", "async", or "exit".
                "async" means checkpoints are persisted async while next graph step executes, replaces checkpoint_during=True
                "sync" means checkpoints are persisted sync after graph step executes, replaces checkpoint_during=False
                "exit" means checkpoints are only persisted when the run exits, does not save intermediate steps

        Returns:
            The created background run.

        ???+ example "Example Usage"

            ```python

            background_run = await client.runs.create(
                thread_id="my_thread_id",
                assistant_id="my_assistant_id",
                input={"messages": [{"role": "user", "content": "hello!"}]},
                metadata={"name":"my_run"},
                context={"model_name": "openai"},
                interrupt_before=["node_to_stop_before_1","node_to_stop_before_2"],
                interrupt_after=["node_to_stop_after_1","node_to_stop_after_2"],
                webhook="https://my.fake.webhook.com",
                multitask_strategy="interrupt"
            )
            print(background_run)
            ```

            ```shell
            --------------------------------------------------------------------------------

            {
                'run_id': 'my_run_id',
                'thread_id': 'my_thread_id',
                'assistant_id': 'my_assistant_id',
                'created_at': '2024-07-25T15:35:42.598503+00:00',
                'updated_at': '2024-07-25T15:35:42.598503+00:00',
                'metadata': {},
                'status': 'pending',
                'kwargs':
                    {
                        'input':
                            {
                                'messages': [
                                    {
                                        'role': 'user',
                                        'content': 'how are you?'
                                    }
                                ]
                            },
                        'config':
                            {
                                'metadata':
                                    {
                                        'created_by': 'system'
                                    },
                                'configurable':
                                    {
                                        'run_id': 'my_run_id',
                                        'user_id': None,
                                        'graph_id': 'agent',
                                        'thread_id': 'my_thread_id',
                                        'checkpoint_id': None,
                                        'assistant_id': 'my_assistant_id'
                                    },
                            },
                        'context':
                            {
                                'model_name': 'openai'
                            }
                        'webhook': "https://my.fake.webhook.com",
                        'temporary': False,
                        'stream_mode': ['values'],
                        'feedback_keys': None,
                        'interrupt_after': ["node_to_stop_after_1","node_to_stop_after_2"],
                        'interrupt_before': ["node_to_stop_before_1","node_to_stop_before_2"]
                    },
                'multitask_strategy': 'interrupt'
            }
            ```
        """  # noqa: E501
        if checkpoint_during is not None:
            warnings.warn(
                "`checkpoint_during` is deprecated and will be removed in a future version. Use `durability` instead.",
                DeprecationWarning,
                stacklevel=2,
            )
        payload = {
            "input": input,
            "command": (
                {k: v for k, v in command.items() if v is not None} if command else None
            ),
            "stream_mode": stream_mode,
            "stream_subgraphs": stream_subgraphs,
            "stream_resumable": stream_resumable,
            "config": config,
            "context": context,
            "metadata": metadata,
            "assistant_id": assistant_id,
            "interrupt_before": interrupt_before,
            "interrupt_after": interrupt_after,
            "webhook": webhook,
            "checkpoint": checkpoint,
            "checkpoint_id": checkpoint_id,
            "checkpoint_during": checkpoint_during,
            "multitask_strategy": multitask_strategy,
            "if_not_exists": if_not_exists,
            "on_completion": on_completion,
            "after_seconds": after_seconds,
            "durability": durability,
        }
        payload = {k: v for k, v in payload.items() if v is not None}

        def on_response(res: httpx.Response):
            """Callback function to handle the response."""
            if on_run_created and (metadata := _get_run_metadata_from_response(res)):
                on_run_created(metadata)

        return await self.http.post(
            f"/threads/{thread_id}/runs" if thread_id else "/runs",
            json=payload,
            params=params,
            headers=headers,
            on_response=on_response if on_run_created else None,
        )

    async def create_batch(
        self,
        payloads: list[RunCreate],
        *,
        headers: Mapping[str, str] | None = None,
        params: QueryParamTypes | None = None,
    ) -> list[Run]:
        """Create a batch of stateless background runs."""

        def filter_payload(payload: RunCreate):
            return {k: v for k, v in payload.items() if v is not None}

        filtered = [filter_payload(payload) for payload in payloads]
        return await self.http.post(
            "/runs/batch", json=filtered, headers=headers, params=params
        )

    @overload
    async def wait(
        self,
        thread_id: str,
        assistant_id: str,
        *,
        input: Mapping[str, Any] | None = None,
        command: Command | None = None,
        metadata: Mapping[str, Any] | None = None,
        config: Config | None = None,
        context: Context | None = None,
        checkpoint: Checkpoint | None = None,
        checkpoint_id: str | None = None,
        checkpoint_during: bool | None = None,
        interrupt_before: All | Sequence[str] | None = None,
        interrupt_after: All | Sequence[str] | None = None,
        webhook: str | None = None,
        on_disconnect: DisconnectMode | None = None,
        multitask_strategy: MultitaskStrategy | None = None,
        if_not_exists: IfNotExists | None = None,
        after_seconds: int | None = None,
        raise_error: bool = True,
        headers: Mapping[str, str] | None = None,
        params: QueryParamTypes | None = None,
        on_run_created: Callable[[RunCreateMetadata], None] | None = None,
    ) -> list[dict] | dict[str, Any]: ...

    @overload
    async def wait(
        self,
        thread_id: None,
        assistant_id: str,
        *,
        input: Mapping[str, Any] | None = None,
        command: Command | None = None,
        metadata: Mapping[str, Any] | None = None,
        config: Config | None = None,
        context: Context | None = None,
        checkpoint_during: bool | None = None,
        interrupt_before: All | Sequence[str] | None = None,
        interrupt_after: All | Sequence[str] | None = None,
        webhook: str | None = None,
        on_disconnect: DisconnectMode | None = None,
        on_completion: OnCompletionBehavior | None = None,
        if_not_exists: IfNotExists | None = None,
        after_seconds: int | None = None,
        raise_error: bool = True,
        headers: Mapping[str, str] | None = None,
        params: QueryParamTypes | None = None,
        on_run_created: Callable[[RunCreateMetadata], None] | None = None,
    ) -> list[dict] | dict[str, Any]: ...

    async def wait(
        self,
        thread_id: str | None,
        assistant_id: str,
        *,
        input: Mapping[str, Any] | None = None,
        command: Command | None = None,
        metadata: Mapping[str, Any] | None = None,
        config: Config | None = None,
        context: Context | None = None,
        checkpoint: Checkpoint | None = None,
        checkpoint_id: str | None = None,
        checkpoint_during: bool | None = None,  # deprecated
        interrupt_before: All | Sequence[str] | None = None,
        interrupt_after: All | Sequence[str] | None = None,
        webhook: str | None = None,
        on_disconnect: DisconnectMode | None = None,
        on_completion: OnCompletionBehavior | None = None,
        multitask_strategy: MultitaskStrategy | None = None,
        if_not_exists: IfNotExists | None = None,
        after_seconds: int | None = None,
        raise_error: bool = True,
        headers: Mapping[str, str] | None = None,
        params: QueryParamTypes | None = None,
        on_run_created: Callable[[RunCreateMetadata], None] | None = None,
        durability: Durability | None = None,
    ) -> list[dict] | dict[str, Any]:
        """Create a run, wait until it finishes and return the final state.

        Args:
            thread_id: the thread ID to create the run on.
                If `None` will create a stateless run.
            assistant_id: The assistant ID or graph name to run.
                If using graph name, will default to first assistant created from that graph.
            input: The input to the graph.
            command: A command to execute. Cannot be combined with input.
            metadata: Metadata to assign to the run.
            config: The configuration for the assistant.
            context: Static context to add to the assistant.
                !!! version-added "Added in version 0.6.0"
            checkpoint: The checkpoint to resume from.
            checkpoint_during: (deprecated) Whether to checkpoint during the run (or only at the end/interruption).
            interrupt_before: Nodes to interrupt immediately before they get executed.
            interrupt_after: Nodes to Nodes to interrupt immediately after they get executed.
            webhook: Webhook to call after LangGraph API call is done.
            on_disconnect: The disconnect mode to use.
                Must be one of 'cancel' or 'continue'.
            on_completion: Whether to delete or keep the thread created for a stateless run.
                Must be one of 'delete' or 'keep'.
            multitask_strategy: Multitask strategy to use.
                Must be one of 'reject', 'interrupt', 'rollback', or 'enqueue'.
            if_not_exists: How to handle missing thread. Defaults to 'reject'.
                Must be either 'reject' (raise error if missing), or 'create' (create new thread).
            after_seconds: The number of seconds to wait before starting the run.
                Use to schedule future runs.
            headers: Optional custom headers to include with the request.
            on_run_created: Optional callback to call when a run is created.
            durability: The durability to use for the run. Values are "sync", "async", or "exit".
                "async" means checkpoints are persisted async while next graph step executes, replaces checkpoint_during=True
                "sync" means checkpoints are persisted sync after graph step executes, replaces checkpoint_during=False
                "exit" means checkpoints are only persisted when the run exits, does not save intermediate steps

        Returns:
            The output of the run.

        ???+ example "Example Usage"

            ```python
            client = get_client(url="http://localhost:2024")
            final_state_of_run = await client.runs.wait(
                thread_id=None,
                assistant_id="agent",
                input={"messages": [{"role": "user", "content": "how are you?"}]},
                metadata={"name":"my_run"},
                context={"model_name": "anthropic"},
                interrupt_before=["node_to_stop_before_1","node_to_stop_before_2"],
                interrupt_after=["node_to_stop_after_1","node_to_stop_after_2"],
                webhook="https://my.fake.webhook.com",
                multitask_strategy="interrupt"
            )
            print(final_state_of_run)
            ```

            ```shell
            -------------------------------------------------------------------------------------------------------------------------------------------

            {
                'messages': [
                    {
                        'content': 'how are you?',
                        'additional_kwargs': {},
                        'response_metadata': {},
                        'type': 'human',
                        'name': None,
                        'id': 'f51a862c-62fe-4866-863b-b0863e8ad78a',
                        'example': False
                    },
                    {
                        'content': "I'm doing well, thanks for asking! I'm an AI assistant created by Anthropic to be helpful, honest, and harmless.",
                        'additional_kwargs': {},
                        'response_metadata': {},
                        'type': 'ai',
                        'name': None,
                        'id': 'run-bf1cd3c6-768f-4c16-b62d-ba6f17ad8b36',
                        'example': False,
                        'tool_calls': [],
                        'invalid_tool_calls': [],
                        'usage_metadata': None
                    }
                ]
            }
            ```

        """  # noqa: E501
        if checkpoint_during is not None:
            warnings.warn(
                "`checkpoint_during` is deprecated and will be removed in a future version. Use `durability` instead.",
                DeprecationWarning,
                stacklevel=2,
            )
        payload = {
            "input": input,
            "command": (
                {k: v for k, v in command.items() if v is not None} if command else None
            ),
            "config": config,
            "context": context,
            "metadata": metadata,
            "assistant_id": assistant_id,
            "interrupt_before": interrupt_before,
            "interrupt_after": interrupt_after,
            "webhook": webhook,
            "checkpoint": checkpoint,
            "checkpoint_id": checkpoint_id,
            "multitask_strategy": multitask_strategy,
            "checkpoint_during": checkpoint_during,
            "if_not_exists": if_not_exists,
            "on_disconnect": on_disconnect,
            "on_completion": on_completion,
            "after_seconds": after_seconds,
            "durability": durability,
        }
        endpoint = (
            f"/threads/{thread_id}/runs/wait" if thread_id is not None else "/runs/wait"
        )

        def on_response(res: httpx.Response):
            """Callback function to handle the response."""
            if on_run_created and (metadata := _get_run_metadata_from_response(res)):
                on_run_created(metadata)

        response = await self.http.request_reconnect(
            endpoint,
            "POST",
            json={k: v for k, v in payload.items() if v is not None},
            params=params,
            headers=headers,
            on_response=on_response if on_run_created else None,
        )
        if (
            raise_error
            and isinstance(response, dict)
            and "__error__" in response
            and isinstance(response["__error__"], dict)
        ):
            raise Exception(
                f"{response['__error__'].get('error')}: {response['__error__'].get('message')}"
            )
        return response

    async def list(
        self,
        thread_id: str,
        *,
        limit: int = 10,
        offset: int = 0,
        status: RunStatus | None = None,
        select: list[RunSelectField] | None = None,
        headers: Mapping[str, str] | None = None,
        params: QueryParamTypes | None = None,
    ) -> list[Run]:
        """List runs.

        Args:
            thread_id: The thread ID to list runs for.
            limit: The maximum number of results to return.
            offset: The number of results to skip.
            status: The status of the run to filter by.
            headers: Optional custom headers to include with the request.
            params: Optional query parameters to include with the request.

        Returns:
            The runs for the thread.

        ???+ example "Example Usage"

            ```python
            client = get_client(url="http://localhost:2024")
            await client.runs.list(
                thread_id="thread_id",
                limit=5,
                offset=5,
            )
            ```

        """  # noqa: E501
        query_params: dict[str, Any] = {
            "limit": limit,
            "offset": offset,
        }
        if status is not None:
            query_params["status"] = status
        if select:
            query_params["select"] = select
        if params:
            query_params.update(params)
        return await self.http.get(
            f"/threads/{thread_id}/runs", params=query_params, headers=headers
        )

    async def get(
        self,
        thread_id: str,
        run_id: str,
        *,
        headers: Mapping[str, str] | None = None,
        params: QueryParamTypes | None = None,
    ) -> Run:
        """Get a run.

        Args:
            thread_id: The thread ID to get.
            run_id: The run ID to get.
            headers: Optional custom headers to include with the request.
            params: Optional query parameters to include with the request.

        Returns:
            `Run` object.

        ???+ example "Example Usage"

            ```python
            client = get_client(url="http://localhost:2024")
            run = await client.runs.get(
                thread_id="thread_id_to_delete",
                run_id="run_id_to_delete",
            )
            ```

        """  # noqa: E501

        return await self.http.get(
            f"/threads/{thread_id}/runs/{run_id}", headers=headers, params=params
        )

    async def cancel(
        self,
        thread_id: str,
        run_id: str,
        *,
        wait: bool = False,
        action: CancelAction = "interrupt",
        headers: Mapping[str, str] | None = None,
        params: QueryParamTypes | None = None,
    ) -> None:
        """Get a run.

        Args:
            thread_id: The thread ID to cancel.
            run_id: The run ID to cancel.
            wait: Whether to wait until run has completed.
            action: Action to take when cancelling the run. Possible values
                are `interrupt` or `rollback`. Default is `interrupt`.
            headers: Optional custom headers to include with the request.
            params: Optional query parameters to include with the request.

        Returns:
            `None`

        ???+ example "Example Usage"

            ```python
            client = get_client(url="http://localhost:2024")
            await client.runs.cancel(
                thread_id="thread_id_to_cancel",
                run_id="run_id_to_cancel",
                wait=True,
                action="interrupt"
            )
            ```

        """  # noqa: E501
        query_params = {
            "wait": 1 if wait else 0,
            "action": action,
        }
        if params:
            query_params.update(params)
        if wait:
            return await self.http.request_reconnect(
                f"/threads/{thread_id}/runs/{run_id}/cancel",
                "POST",
                params=query_params,
                headers=headers,
            )
        else:
            return await self.http.post(
                f"/threads/{thread_id}/runs/{run_id}/cancel",
                json=None,
                params=query_params,
                headers=headers,
            )

    async def join(
        self,
        thread_id: str,
        run_id: str,
        *,
        headers: Mapping[str, str] | None = None,
        params: QueryParamTypes | None = None,
    ) -> dict:
        """Block until a run is done. Returns the final state of the thread.

        Args:
            thread_id: The thread ID to join.
            run_id: The run ID to join.
            headers: Optional custom headers to include with the request.
            params: Optional query parameters to include with the request.

        Returns:
            `None`

        ???+ example "Example Usage"

            ```python
            client = get_client(url="http://localhost:2024")
            result =await client.runs.join(
                thread_id="thread_id_to_join",
                run_id="run_id_to_join"
            )
            ```

        """  # noqa: E501
        return await self.http.request_reconnect(
            f"/threads/{thread_id}/runs/{run_id}/join",
            "GET",
            headers=headers,
            params=params,
        )

    def join_stream(
        self,
        thread_id: str,
        run_id: str,
        *,
        cancel_on_disconnect: bool = False,
        stream_mode: StreamMode | Sequence[StreamMode] | None = None,
        headers: Mapping[str, str] | None = None,
        params: QueryParamTypes | None = None,
        last_event_id: str | None = None,
    ) -> AsyncIterator[StreamPart]:
        """Stream output from a run in real-time, until the run is done.
        Output is not buffered, so any output produced before this call will
        not be received here.

        Args:
            thread_id: The thread ID to join.
            run_id: The run ID to join.
            cancel_on_disconnect: Whether to cancel the run when the stream is disconnected.
            stream_mode: The stream mode(s) to use. Must be a subset of the stream modes passed
                when creating the run. Background runs default to having the union of all
                stream modes.
            headers: Optional custom headers to include with the request.
            params: Optional query parameters to include with the request.
            last_event_id: The last event ID to use for the stream.

        Returns:
            The stream of parts.

        ???+ example "Example Usage"

            ```python
            client = get_client(url="http://localhost:2024")
            async for part in client.runs.join_stream(
                thread_id="thread_id_to_join",
                run_id="run_id_to_join",
                stream_mode=["values", "debug"]
            ):
                print(part)
            ```

        """  # noqa: E501
        query_params = {
            "cancel_on_disconnect": cancel_on_disconnect,
            "stream_mode": stream_mode,
        }
        if params:
            query_params.update(params)
        return self.http.stream(
            f"/threads/{thread_id}/runs/{run_id}/stream",
            "GET",
            params=query_params,
            headers={
                **({"Last-Event-ID": last_event_id} if last_event_id else {}),
                **(headers or {}),
            }
            or None,
        )

    async def delete(
        self,
        thread_id: str,
        run_id: str,
        *,
        headers: Mapping[str, str] | None = None,
        params: QueryParamTypes | None = None,
    ) -> None:
        """Delete a run.

        Args:
            thread_id: The thread ID to delete.
            run_id: The run ID to delete.
            headers: Optional custom headers to include with the request.
            params: Optional query parameters to include with the request.

        Returns:
            `None`

        ???+ example "Example Usage"

            ```python
            client = get_client(url="http://localhost:2024")
            await client.runs.delete(
                thread_id="thread_id_to_delete",
                run_id="run_id_to_delete"
            )
            ```

        """  # noqa: E501
        await self.http.delete(
            f"/threads/{thread_id}/runs/{run_id}", headers=headers, params=params
        )


class CronClient:
    """Client for managing recurrent runs (cron jobs) in LangGraph.

    A run is a single invocation of an assistant with optional input, config, and context.
    This client allows scheduling recurring runs to occur automatically.

    ???+ example "Example Usage"

        ```python
        client = get_client(url="http://localhost:2024"))
        cron_job = await client.crons.create_for_thread(
            thread_id="thread_123",
            assistant_id="asst_456",
            schedule="0 9 * * *",
            input={"message": "Daily update"}
        )
        ```

    !!! note "Feature Availability"

        The crons client functionality is not supported on all licenses.
        Please check the relevant license documentation for the most up-to-date
        details on feature availability.
    """

    def __init__(self, http_client: HttpClient) -> None:
        self.http = http_client

    async def create_for_thread(
        self,
        thread_id: str,
        assistant_id: str,
        *,
        schedule: str,
        input: Mapping[str, Any] | None = None,
        metadata: Mapping[str, Any] | None = None,
        config: Config | None = None,
        context: Context | None = None,
        checkpoint_during: bool | None = None,
        interrupt_before: All | list[str] | None = None,
        interrupt_after: All | list[str] | None = None,
        webhook: str | None = None,
        multitask_strategy: str | None = None,
        headers: Mapping[str, str] | None = None,
        params: QueryParamTypes | None = None,
    ) -> Run:
        """Create a cron job for a thread.

        Args:
            thread_id: the thread ID to run the cron job on.
            assistant_id: The assistant ID or graph name to use for the cron job.
                If using graph name, will default to first assistant created from that graph.
            schedule: The cron schedule to execute this job on.
            input: The input to the graph.
            metadata: Metadata to assign to the cron job runs.
            config: The configuration for the assistant.
            context: Static context to add to the assistant.
                !!! version-added "Added in version 0.6.0"
            checkpoint_during: Whether to checkpoint during the run (or only at the end/interruption).
            interrupt_before: Nodes to interrupt immediately before they get executed.

            interrupt_after: Nodes to Nodes to interrupt immediately after they get executed.

            webhook: Webhook to call after LangGraph API call is done.
            multitask_strategy: Multitask strategy to use.
                Must be one of 'reject', 'interrupt', 'rollback', or 'enqueue'.
            headers: Optional custom headers to include with the request.
            params: Optional query parameters to include with the request.

        Returns:
            The cron run.

        ???+ example "Example Usage"

            ```python
            client = get_client(url="http://localhost:2024")
            cron_run = await client.crons.create_for_thread(
                thread_id="my-thread-id",
                assistant_id="agent",
                schedule="27 15 * * *",
                input={"messages": [{"role": "user", "content": "hello!"}]},
                metadata={"name":"my_run"},
                context={"model_name": "openai"},
                interrupt_before=["node_to_stop_before_1","node_to_stop_before_2"],
                interrupt_after=["node_to_stop_after_1","node_to_stop_after_2"],
                webhook="https://my.fake.webhook.com",
                multitask_strategy="interrupt"
            )
            ```
        """  # noqa: E501
        payload = {
            "schedule": schedule,
            "input": input,
            "config": config,
            "metadata": metadata,
            "context": context,
            "assistant_id": assistant_id,
            "checkpoint_during": checkpoint_during,
            "interrupt_before": interrupt_before,
            "interrupt_after": interrupt_after,
            "webhook": webhook,
        }
        if multitask_strategy:
            payload["multitask_strategy"] = multitask_strategy
        payload = {k: v for k, v in payload.items() if v is not None}
        return await self.http.post(
            f"/threads/{thread_id}/runs/crons",
            json=payload,
            headers=headers,
            params=params,
        )

    async def create(
        self,
        assistant_id: str,
        *,
        schedule: str,
        input: Mapping[str, Any] | None = None,
        metadata: Mapping[str, Any] | None = None,
        config: Config | None = None,
        context: Context | None = None,
        checkpoint_during: bool | None = None,
        interrupt_before: All | list[str] | None = None,
        interrupt_after: All | list[str] | None = None,
        webhook: str | None = None,
        multitask_strategy: str | None = None,
        headers: Mapping[str, str] | None = None,
        params: QueryParamTypes | None = None,
    ) -> Run:
        """Create a cron run.

        Args:
            assistant_id: The assistant ID or graph name to use for the cron job.
                If using graph name, will default to first assistant created from that graph.
            schedule: The cron schedule to execute this job on.
            input: The input to the graph.
            metadata: Metadata to assign to the cron job runs.
            config: The configuration for the assistant.
            context: Static context to add to the assistant.
                !!! version-added "Added in version 0.6.0"
            checkpoint_during: Whether to checkpoint during the run (or only at the end/interruption).
            interrupt_before: Nodes to interrupt immediately before they get executed.
            interrupt_after: Nodes to Nodes to interrupt immediately after they get executed.
            webhook: Webhook to call after LangGraph API call is done.
            multitask_strategy: Multitask strategy to use.
                Must be one of 'reject', 'interrupt', 'rollback', or 'enqueue'.
            headers: Optional custom headers to include with the request.
            params: Optional query parameters to include with the request.

        Returns:
            The cron run.

        ???+ example "Example Usage"

            ```python
            client = get_client(url="http://localhost:2024")
            cron_run = client.crons.create(
                assistant_id="agent",
                schedule="27 15 * * *",
                input={"messages": [{"role": "user", "content": "hello!"}]},
                metadata={"name":"my_run"},
                context={"model_name": "openai"},
                interrupt_before=["node_to_stop_before_1","node_to_stop_before_2"],
                interrupt_after=["node_to_stop_after_1","node_to_stop_after_2"],
                webhook="https://my.fake.webhook.com",
                multitask_strategy="interrupt"
            )
            ```

        """  # noqa: E501
        payload = {
            "schedule": schedule,
            "input": input,
            "config": config,
            "metadata": metadata,
            "context": context,
            "assistant_id": assistant_id,
            "checkpoint_during": checkpoint_during,
            "interrupt_before": interrupt_before,
            "interrupt_after": interrupt_after,
            "webhook": webhook,
        }
        if multitask_strategy:
            payload["multitask_strategy"] = multitask_strategy
        payload = {k: v for k, v in payload.items() if v is not None}
        return await self.http.post(
            "/runs/crons", json=payload, headers=headers, params=params
        )

    async def delete(
        self,
        cron_id: str,
        *,
        headers: Mapping[str, str] | None = None,
        params: QueryParamTypes | None = None,
    ) -> None:
        """Delete a cron.

        Args:
            cron_id: The cron ID to delete.
            headers: Optional custom headers to include with the request.
            params: Optional query parameters to include with the request.

        Returns:
            `None`

        ???+ example "Example Usage"

            ```python
            client = get_client(url="http://localhost:2024")
            await client.crons.delete(
                cron_id="cron_to_delete"
            )
            ```

        """  # noqa: E501
        await self.http.delete(f"/runs/crons/{cron_id}", headers=headers, params=params)

    async def search(
        self,
        *,
        assistant_id: str | None = None,
        thread_id: str | None = None,
        limit: int = 10,
        offset: int = 0,
        sort_by: CronSortBy | None = None,
        sort_order: SortOrder | None = None,
        select: list[CronSelectField] | None = None,
        headers: Mapping[str, str] | None = None,
        params: QueryParamTypes | None = None,
    ) -> list[Cron]:
        """Get a list of cron jobs.

        Args:
            assistant_id: The assistant ID or graph name to search for.
            thread_id: the thread ID to search for.
            limit: The maximum number of results to return.
            offset: The number of results to skip.
            headers: Optional custom headers to include with the request.
            params: Optional query parameters to include with the request.

        Returns:
            The list of cron jobs returned by the search,

        ???+ example "Example Usage"

            ```python
            client = get_client(url="http://localhost:2024")
            cron_jobs = await client.crons.search(
                assistant_id="my_assistant_id",
                thread_id="my_thread_id",
                limit=5,
                offset=5,
            )
            print(cron_jobs)
            ```
            ```shell

            ----------------------------------------------------------

            [
                {
                    'cron_id': '1ef3cefa-4c09-6926-96d0-3dc97fd5e39b',
                    'assistant_id': 'my_assistant_id',
                    'thread_id': 'my_thread_id',
                    'user_id': None,
                    'payload':
                        {
                            'input': {'start_time': ''},
                            'schedule': '4 * * * *',
                            'assistant_id': 'my_assistant_id'
                        },
                    'schedule': '4 * * * *',
                    'next_run_date': '2024-07-25T17:04:00+00:00',
                    'end_time': None,
                    'created_at': '2024-07-08T06:02:23.073257+00:00',
                    'updated_at': '2024-07-08T06:02:23.073257+00:00'
                }
            ]
            ```

        """  # noqa: E501
        payload = {
            "assistant_id": assistant_id,
            "thread_id": thread_id,
            "limit": limit,
            "offset": offset,
        }
        if sort_by:
            payload["sort_by"] = sort_by
        if sort_order:
            payload["sort_order"] = sort_order
        if select:
            payload["select"] = select
        payload = {k: v for k, v in payload.items() if v is not None}
        return await self.http.post(
            "/runs/crons/search", json=payload, headers=headers, params=params
        )

    async def count(
        self,
        *,
        assistant_id: str | None = None,
        thread_id: str | None = None,
        headers: Mapping[str, str] | None = None,
        params: QueryParamTypes | None = None,
    ) -> int:
        """Count cron jobs matching filters.

        Args:
            assistant_id: Assistant ID to filter by.
            thread_id: Thread ID to filter by.
            headers: Optional custom headers to include with the request.
            params: Optional query parameters to include with the request.

        Returns:
            int: Number of crons matching the criteria.
        """
        payload: dict[str, Any] = {}
        if assistant_id:
            payload["assistant_id"] = assistant_id
        if thread_id:
            payload["thread_id"] = thread_id
        return await self.http.post(
            "/runs/crons/count", json=payload, headers=headers, params=params
        )


class StoreClient:
    """Client for interacting with the graph's shared storage.

    The Store provides a key-value storage system for persisting data across graph executions,
    allowing for stateful operations and data sharing across threads.

    ???+ example "Example"

        ```python
        client = get_client(url="http://localhost:2024")
        await client.store.put_item(["users", "user123"], "mem-123451342", {"name": "Alice", "score": 100})
        ```
    """

    def __init__(self, http: HttpClient) -> None:
        self.http = http

    async def put_item(
        self,
        namespace: Sequence[str],
        /,
        key: str,
        value: Mapping[str, Any],
        index: Literal[False] | list[str] | None = None,
        ttl: int | None = None,
        headers: Mapping[str, str] | None = None,
        params: QueryParamTypes | None = None,
    ) -> None:
        """Store or update an item.

        Args:
            namespace: A list of strings representing the namespace path.
            key: The unique identifier for the item within the namespace.
            value: A dictionary containing the item's data.
            index: Controls search indexing - None (use defaults), False (disable), or list of field paths to index.
            ttl: Optional time-to-live in minutes for the item, or None for no expiration.
            headers: Optional custom headers to include with the request.
            params: Optional query parameters to include with the request.

        Returns:
            `None`

        ???+ example "Example Usage"

            ```python
            client = get_client(url="http://localhost:2024")
            await client.store.put_item(
                ["documents", "user123"],
                key="item456",
                value={"title": "My Document", "content": "Hello World"}
            )
            ```
        """
        for label in namespace:
            if "." in label:
                raise ValueError(
                    f"Invalid namespace label '{label}'. Namespace labels cannot contain periods ('.')."
                )
        payload = {
            "namespace": namespace,
            "key": key,
            "value": value,
            "index": index,
            "ttl": ttl,
        }
        await self.http.put(
            "/store/items", json=_provided_vals(payload), headers=headers, params=params
        )

    async def get_item(
        self,
        namespace: Sequence[str],
        /,
        key: str,
        *,
        refresh_ttl: bool | None = None,
        headers: Mapping[str, str] | None = None,
        params: QueryParamTypes | None = None,
    ) -> Item:
        """Retrieve a single item.

        Args:
            key: The unique identifier for the item.
            namespace: Optional list of strings representing the namespace path.
            refresh_ttl: Whether to refresh the TTL on this read operation. If `None`, uses the store's default behavior.

        Returns:
            Item: The retrieved item.
            headers: Optional custom headers to include with the request.
            params: Optional query parameters to include with the request.

        ???+ example "Example Usage"

            ```python
            client = get_client(url="http://localhost:2024")
            item = await client.store.get_item(
                ["documents", "user123"],
                key="item456",
            )
            print(item)
            ```
            ```shell

            ----------------------------------------------------------------

            {
                'namespace': ['documents', 'user123'],
                'key': 'item456',
                'value': {'title': 'My Document', 'content': 'Hello World'},
                'created_at': '2024-07-30T12:00:00Z',
                'updated_at': '2024-07-30T12:00:00Z'
            }
            ```
        """
        for label in namespace:
            if "." in label:
                raise ValueError(
                    f"Invalid namespace label '{label}'. Namespace labels cannot contain periods ('.')."
                )
        get_params = {"namespace": ".".join(namespace), "key": key}
        if refresh_ttl is not None:
            get_params["refresh_ttl"] = refresh_ttl
        if params:
            get_params = {**get_params, **params}
        return await self.http.get("/store/items", params=get_params, headers=headers)

    async def delete_item(
        self,
        namespace: Sequence[str],
        /,
        key: str,
        headers: Mapping[str, str] | None = None,
        params: QueryParamTypes | None = None,
    ) -> None:
        """Delete an item.

        Args:
            key: The unique identifier for the item.
            namespace: Optional list of strings representing the namespace path.
            headers: Optional custom headers to include with the request.
            params: Optional query parameters to include with the request.

        Returns:
            `None`

        ???+ example "Example Usage"

            ```python
            client = get_client(url="http://localhost:2024")
            await client.store.delete_item(
                ["documents", "user123"],
                key="item456",
            )
            ```
        """
        await self.http.delete(
            "/store/items",
            json={"namespace": namespace, "key": key},
            headers=headers,
            params=params,
        )

    async def search_items(
        self,
        namespace_prefix: Sequence[str],
        /,
        filter: Mapping[str, Any] | None = None,
        limit: int = 10,
        offset: int = 0,
        query: str | None = None,
        refresh_ttl: bool | None = None,
        headers: Mapping[str, str] | None = None,
        params: QueryParamTypes | None = None,
    ) -> SearchItemsResponse:
        """Search for items within a namespace prefix.

        Args:
            namespace_prefix: List of strings representing the namespace prefix.
            filter: Optional dictionary of key-value pairs to filter results.
            limit: Maximum number of items to return (default is 10).
            offset: Number of items to skip before returning results (default is 0).
            query: Optional query for natural language search.
            refresh_ttl: Whether to refresh the TTL on items returned by this search. If `None`, uses the store's default behavior.
            headers: Optional custom headers to include with the request.
            params: Optional query parameters to include with the request.

        Returns:
            A list of items matching the search criteria.

        ???+ example "Example Usage"

            ```python
            client = get_client(url="http://localhost:2024")
            items = await client.store.search_items(
                ["documents"],
                filter={"author": "John Doe"},
                limit=5,
                offset=0
            )
            print(items)
            ```
            ```shell

            ----------------------------------------------------------------

            {
                "items": [
                    {
                        "namespace": ["documents", "user123"],
                        "key": "item789",
                        "value": {
                            "title": "Another Document",
                            "author": "John Doe"
                        },
                        "created_at": "2024-07-30T12:00:00Z",
                        "updated_at": "2024-07-30T12:00:00Z"
                    },
                    # ... additional items ...
                ]
            }
            ```
        """
        payload = {
            "namespace_prefix": namespace_prefix,
            "filter": filter,
            "limit": limit,
            "offset": offset,
            "query": query,
            "refresh_ttl": refresh_ttl,
        }

        return await self.http.post(
            "/store/items/search",
            json=_provided_vals(payload),
            headers=headers,
            params=params,
        )

    async def list_namespaces(
        self,
        prefix: list[str] | None = None,
        suffix: list[str] | None = None,
        max_depth: int | None = None,
        limit: int = 100,
        offset: int = 0,
        headers: Mapping[str, str] | None = None,
        params: QueryParamTypes | None = None,
    ) -> ListNamespaceResponse:
        """List namespaces with optional match conditions.

        Args:
            prefix: Optional list of strings representing the prefix to filter namespaces.
            suffix: Optional list of strings representing the suffix to filter namespaces.
            max_depth: Optional integer specifying the maximum depth of namespaces to return.
            limit: Maximum number of namespaces to return (default is 100).
            offset: Number of namespaces to skip before returning results (default is 0).
            headers: Optional custom headers to include with the request.
            params: Optional query parameters to include with the request.

        Returns:
            A list of namespaces matching the criteria.

        ???+ example "Example Usage"

            ```python
            client = get_client(url="http://localhost:2024")
            namespaces = await client.store.list_namespaces(
                prefix=["documents"],
                max_depth=3,
                limit=10,
                offset=0
            )
            print(namespaces)

            ----------------------------------------------------------------

            [
                ["documents", "user123", "reports"],
                ["documents", "user456", "invoices"],
                ...
            ]
            ```
        """
        payload = {
            "prefix": prefix,
            "suffix": suffix,
            "max_depth": max_depth,
            "limit": limit,
            "offset": offset,
        }
        return await self.http.post(
            "/store/namespaces",
            json=_provided_vals(payload),
            headers=headers,
            params=params,
        )


def get_sync_client(
    *,
    url: str | None = None,
    api_key: str | None = None,
    headers: Mapping[str, str] | None = None,
    timeout: TimeoutTypes | None = None,
) -> SyncLangGraphClient:
    """Get a synchronous LangGraphClient instance.

    Args:
        url: The URL of the LangGraph API.
        api_key: The API key. If not provided, it will be read from the environment.
            Precedence:
                1. explicit argument
                2. LANGGRAPH_API_KEY
                3. LANGSMITH_API_KEY
                4. LANGCHAIN_API_KEY
        headers: Optional custom headers
        timeout: Optional timeout configuration for the HTTP client.
            Accepts an httpx.Timeout instance, a float (seconds), or a tuple of timeouts.
            Tuple format is (connect, read, write, pool)
            If not provided, defaults to connect=5s, read=300s, write=300s, and pool=5s.
    Returns:
        SyncLangGraphClient: The top-level synchronous client for accessing AssistantsClient,
        ThreadsClient, RunsClient, and CronClient.

    ???+ example "Example"

        ```python
        from langgraph_sdk import get_sync_client

        # get top-level synchronous LangGraphClient
        client = get_sync_client(url="http://localhost:8123")

        # example usage: client.<model>.<method_name>()
        assistant = client.assistants.get(assistant_id="some_uuid")
        ```
    """

    if url is None:
        url = "http://localhost:8123"

    transport = httpx.HTTPTransport(retries=5)
    client = httpx.Client(
        base_url=url,
        transport=transport,
        timeout=(
            httpx.Timeout(timeout)
            if timeout is not None
            else httpx.Timeout(connect=5, read=300, write=300, pool=5)
        ),
        headers=_get_headers(api_key, headers),
    )
    return SyncLangGraphClient(client)


class SyncLangGraphClient:
    """Synchronous client for interacting with the LangGraph API.

    This class provides synchronous access to LangGraph API endpoints for managing
    assistants, threads, runs, cron jobs, and data storage.

    ???+ example "Example"

        ```python
        client = get_sync_client(url="http://localhost:2024")
        assistant = client.assistants.get("asst_123")
        ```
    """

    def __init__(self, client: httpx.Client) -> None:
        self.http = SyncHttpClient(client)
        self.assistants = SyncAssistantsClient(self.http)
        self.threads = SyncThreadsClient(self.http)
        self.runs = SyncRunsClient(self.http)
        self.crons = SyncCronClient(self.http)
        self.store = SyncStoreClient(self.http)

    def __enter__(self) -> SyncLangGraphClient:
        """Enter the sync context manager."""
        return self

    def __exit__(
        self,
        exc_type: type[BaseException] | None,
        exc_val: BaseException | None,
        exc_tb: TracebackType | None,
    ) -> None:
        """Exit the sync context manager."""
        self.close()

    def close(self) -> None:
        """Close the underlying HTTP client."""
        if hasattr(self, "http"):
            self.http.client.close()


class SyncHttpClient:
    """Handle synchronous requests to the LangGraph API.

    Provides error messaging and content handling enhancements above the
    underlying httpx client, mirroring the interface of [HttpClient](#HttpClient)
    but for sync usage.

    Attributes:
        client (httpx.Client): Underlying HTTPX sync client.
    """

    def __init__(self, client: httpx.Client) -> None:
        self.client = client

    def get(
        self,
        path: str,
        *,
        params: QueryParamTypes | None = None,
        headers: Mapping[str, str] | None = None,
        on_response: Callable[[httpx.Response], None] | None = None,
    ) -> Any:
        """Send a `GET` request."""
        r = self.client.get(path, params=params, headers=headers)
        if on_response:
            on_response(r)
        _raise_for_status_typed(r)
        return _decode_json(r)

    def post(
        self,
        path: str,
        *,
        json: dict[str, Any] | list | None,
        params: QueryParamTypes | None = None,
        headers: Mapping[str, str] | None = None,
        on_response: Callable[[httpx.Response], None] | None = None,
    ) -> Any:
        """Send a `POST` request."""
        if json is not None:
            request_headers, content = _encode_json(json)
        else:
            request_headers, content = {}, b""
        if headers:
            request_headers.update(headers)
        r = self.client.post(
            path, headers=request_headers, content=content, params=params
        )
        if on_response:
            on_response(r)
        _raise_for_status_typed(r)
        return _decode_json(r)

    def put(
        self,
        path: str,
        *,
        json: dict,
        params: QueryParamTypes | None = None,
        headers: Mapping[str, str] | None = None,
        on_response: Callable[[httpx.Response], None] | None = None,
    ) -> Any:
        """Send a `PUT` request."""
        request_headers, content = _encode_json(json)
        if headers:
            request_headers.update(headers)

        r = self.client.put(
            path, headers=request_headers, content=content, params=params
        )
        if on_response:
            on_response(r)
        _raise_for_status_typed(r)
        return _decode_json(r)

    def patch(
        self,
        path: str,
        *,
        json: dict,
        params: QueryParamTypes | None = None,
        headers: Mapping[str, str] | None = None,
        on_response: Callable[[httpx.Response], None] | None = None,
    ) -> Any:
        """Send a `PATCH` request."""
        request_headers, content = _encode_json(json)
        if headers:
            request_headers.update(headers)
        r = self.client.patch(
            path, headers=request_headers, content=content, params=params
        )
        if on_response:
            on_response(r)
        _raise_for_status_typed(r)
        return _decode_json(r)

    def delete(
        self,
        path: str,
        *,
        json: Any | None = None,
        params: QueryParamTypes | None = None,
        headers: Mapping[str, str] | None = None,
        on_response: Callable[[httpx.Response], None] | None = None,
    ) -> None:
        """Send a `DELETE` request."""
        r = self.client.request(
            "DELETE", path, json=json, params=params, headers=headers
        )
        if on_response:
            on_response(r)
        _raise_for_status_typed(r)

    def request_reconnect(
        self,
        path: str,
        method: str,
        *,
        json: dict[str, Any] | None = None,
        params: QueryParamTypes | None = None,
        headers: Mapping[str, str] | None = None,
        on_response: Callable[[httpx.Response], None] | None = None,
        reconnect_limit: int = 5,
    ) -> Any:
        """Send a request that automatically reconnects to Location header."""
        request_headers, content = _encode_json(json)
        if headers:
            request_headers.update(headers)
        with self.client.stream(
            method, path, headers=request_headers, content=content, params=params
        ) as r:
            if on_response:
                on_response(r)
            try:
                r.raise_for_status()
            except httpx.HTTPStatusError as e:
                body = r.read().decode()
                if sys.version_info >= (3, 11):
                    e.add_note(body)
                else:
                    logger.error(f"Error from langgraph-api: {body}", exc_info=e)
                raise e
            loc = r.headers.get("location")
            if reconnect_limit <= 0 or not loc:
                return _decode_json(r)
            try:
                return _decode_json(r)
            except httpx.HTTPError:
                warnings.warn(
                    f"Request failed, attempting reconnect to Location: {loc}",
                    stacklevel=2,
                )
                r.close()
                return self.request_reconnect(
                    loc,
                    "GET",
                    headers=request_headers,
                    # don't pass on_response so it's only called once
                    reconnect_limit=reconnect_limit - 1,
                )

    def stream(
        self,
        path: str,
        method: str,
        *,
        json: dict[str, Any] | None = None,
        params: QueryParamTypes | None = None,
        headers: Mapping[str, str] | None = None,
        on_response: Callable[[httpx.Response], None] | None = None,
    ) -> Iterator[StreamPart]:
        """Stream the results of a request using SSE."""
        if json is not None:
            request_headers, content = _encode_json(json)
        else:
            request_headers, content = {}, None
        request_headers["Accept"] = "text/event-stream"
        request_headers["Cache-Control"] = "no-store"
        if headers:
            request_headers.update(headers)

        reconnect_headers = {
            key: value
            for key, value in request_headers.items()
            if key.lower() not in {"content-length", "content-type"}
        }

        last_event_id: str | None = None
        reconnect_path: str | None = None
        reconnect_attempts = 0
        max_reconnect_attempts = 5

        while True:
            current_headers = dict(
                request_headers if reconnect_path is None else reconnect_headers
            )
            if last_event_id is not None:
                current_headers["Last-Event-ID"] = last_event_id

            current_method = method if reconnect_path is None else "GET"
            current_content = content if reconnect_path is None else None
            current_params = params if reconnect_path is None else None

            retry = False
            with self.client.stream(
                current_method,
                reconnect_path or path,
                headers=current_headers,
                content=current_content,
                params=current_params,
            ) as res:
                if reconnect_path is None and on_response:
                    on_response(res)
                # check status
                _raise_for_status_typed(res)
                # check content type
                content_type = res.headers.get("content-type", "").partition(";")[0]
                if "text/event-stream" not in content_type:
                    raise httpx.TransportError(
                        "Expected response header Content-Type to contain 'text/event-stream', "
                        f"got {content_type!r}"
                    )

                reconnect_location = res.headers.get("location")
                if reconnect_location:
                    reconnect_path = reconnect_location

                decoder = SSEDecoder()
                try:
                    for line in iter_lines_raw(res):
                        sse = decoder.decode(line.rstrip(b"\n"))
                        if sse is not None:
                            if decoder.last_event_id is not None:
                                last_event_id = decoder.last_event_id
                            if sse.event or sse.data is not None:
                                yield sse
                except httpx.HTTPError:
                    # httpx.TransportError inherits from HTTPError, so transient
                    # disconnects during streaming land here.
                    if reconnect_path is None:
                        raise
                    retry = True
                else:
                    if sse := decoder.decode(b""):
                        if decoder.last_event_id is not None:
                            last_event_id = decoder.last_event_id
                        if sse.event or sse.data is not None:
                            # See async stream implementation for rationale on
                            # skipping empty flush events.
                            yield sse
            if retry:
                reconnect_attempts += 1
                if reconnect_attempts > max_reconnect_attempts:
                    raise httpx.TransportError(
                        "Exceeded maximum SSE reconnection attempts"
                    )
                continue
            break


def _encode_json(json: Any) -> tuple[dict[str, str], bytes]:
    body = orjson.dumps(
        json,
        _orjson_default,
        orjson.OPT_SERIALIZE_NUMPY | orjson.OPT_NON_STR_KEYS,
    )
    content_length = str(len(body))
    content_type = "application/json"
    headers = {"Content-Length": content_length, "Content-Type": content_type}
    return headers, body


def _decode_json(r: httpx.Response) -> Any:
    body = r.read()
    return orjson.loads(body) if body else None


class SyncAssistantsClient:
    """Client for managing assistants in LangGraph synchronously.

    This class provides methods to interact with assistants, which are versioned configurations of your graph.

    ???+ example "Example"

        ```python
        client = get_sync_client(url="http://localhost:2024")
        assistant = client.assistants.get("assistant_id_123")
        ```
    """

    def __init__(self, http: SyncHttpClient) -> None:
        self.http = http

    def get(
        self,
        assistant_id: str,
        *,
        headers: Mapping[str, str] | None = None,
        params: QueryParamTypes | None = None,
    ) -> Assistant:
        """Get an assistant by ID.

        Args:
            assistant_id: The ID of the assistant to get OR the name of the graph (to use the default assistant).
            headers: Optional custom headers to include with the request.
            params: Optional query parameters to include with the request.

        Returns:
            `Assistant` Object.

        ???+ example "Example Usage"

            ```python
            assistant = client.assistants.get(
                assistant_id="my_assistant_id"
            )
            print(assistant)
            ```

            ```shell
            ----------------------------------------------------

            {
                'assistant_id': 'my_assistant_id',
                'graph_id': 'agent',
                'created_at': '2024-06-25T17:10:33.109781+00:00',
                'updated_at': '2024-06-25T17:10:33.109781+00:00',
                'config': {},
                'context': {},
                'metadata': {'created_by': 'system'}
            }
            ```

        """  # noqa: E501
        return self.http.get(
            f"/assistants/{assistant_id}", headers=headers, params=params
        )

    def get_graph(
        self,
        assistant_id: str,
        *,
        xray: int | bool = False,
        headers: Mapping[str, str] | None = None,
        params: QueryParamTypes | None = None,
    ) -> dict[str, list[dict[str, Any]]]:
        """Get the graph of an assistant by ID.

        Args:
            assistant_id: The ID of the assistant to get the graph of.
            xray: Include graph representation of subgraphs. If an integer value is provided, only subgraphs with a depth less than or equal to the value will be included.
            headers: Optional custom headers to include with the request.
            params: Optional query parameters to include with the request.

        Returns:
            The graph information for the assistant in JSON format.

        ???+ example "Example Usage"

            ```python
            client = get_sync_client(url="http://localhost:2024")
            graph_info = client.assistants.get_graph(
                assistant_id="my_assistant_id"
            )
            print(graph_info)

            --------------------------------------------------------------------------------------------------------------------------

            {
                'nodes':
                    [
                        {'id': '__start__', 'type': 'schema', 'data': '__start__'},
                        {'id': '__end__', 'type': 'schema', 'data': '__end__'},
                        {'id': 'agent','type': 'runnable','data': {'id': ['langgraph', 'utils', 'RunnableCallable'],'name': 'agent'}},
                    ],
                'edges':
                    [
                        {'source': '__start__', 'target': 'agent'},
                        {'source': 'agent','target': '__end__'}
                    ]
            }
            ```

        """  # noqa: E501
        query_params = {"xray": xray}
        if params:
            query_params.update(params)
        return self.http.get(
            f"/assistants/{assistant_id}/graph", params=query_params, headers=headers
        )

    def get_schemas(
        self,
        assistant_id: str,
        *,
        headers: Mapping[str, str] | None = None,
        params: QueryParamTypes | None = None,
    ) -> GraphSchema:
        """Get the schemas of an assistant by ID.

        Args:
            assistant_id: The ID of the assistant to get the schema of.
            headers: Optional custom headers to include with the request.
            params: Optional query parameters to include with the request.

        Returns:
            GraphSchema: The graph schema for the assistant.

        ???+ example "Example Usage"

            ```python
            client = get_sync_client(url="http://localhost:2024")
            schema = client.assistants.get_schemas(
                assistant_id="my_assistant_id"
            )
            print(schema)
            ```
            ```shell
            ----------------------------------------------------------------------------------------------------------------------------

            {
                'graph_id': 'agent',
                'state_schema':
                    {
                        'title': 'LangGraphInput',
                        '$ref': '#/definitions/AgentState',
                        'definitions':
                            {
                                'BaseMessage':
                                    {
                                        'title': 'BaseMessage',
                                        'description': 'Base abstract Message class. Messages are the inputs and outputs of ChatModels.',
                                        'type': 'object',
                                        'properties':
                                            {
                                             'content':
                                                {
                                                    'title': 'Content',
                                                    'anyOf': [
                                                        {'type': 'string'},
                                                        {'type': 'array','items': {'anyOf': [{'type': 'string'}, {'type': 'object'}]}}
                                                    ]
                                                },
                                            'additional_kwargs':
                                                {
                                                    'title': 'Additional Kwargs',
                                                    'type': 'object'
                                                },
                                            'response_metadata':
                                                {
                                                    'title': 'Response Metadata',
                                                    'type': 'object'
                                                },
                                            'type':
                                                {
                                                    'title': 'Type',
                                                    'type': 'string'
                                                },
                                            'name':
                                                {
                                                    'title': 'Name',
                                                    'type': 'string'
                                                },
                                            'id':
                                                {
                                                    'title': 'Id',
                                                    'type': 'string'
                                                }
                                            },
                                        'required': ['content', 'type']
                                    },
                                'AgentState':
                                    {
                                        'title': 'AgentState',
                                        'type': 'object',
                                        'properties':
                                            {
                                                'messages':
                                                    {
                                                        'title': 'Messages',
                                                        'type': 'array',
                                                        'items': {'$ref': '#/definitions/BaseMessage'}
                                                    }
                                            },
                                        'required': ['messages']
                                    }
                            }
                    },
                'config_schema':
                    {
                        'title': 'Configurable',
                        'type': 'object',
                        'properties':
                            {
                                'model_name':
                                    {
                                        'title': 'Model Name',
                                        'enum': ['anthropic', 'openai'],
                                        'type': 'string'
                                    }
                            }
                    },
                'context_schema':
                    {
                        'title': 'Context',
                        'type': 'object',
                        'properties':
                            {
                                'model_name':
                                    {
                                        'title': 'Model Name',
                                        'enum': ['anthropic', 'openai'],
                                        'type': 'string'
                                    }
                            }
                    }
            }
            ```

        """  # noqa: E501
        return self.http.get(
            f"/assistants/{assistant_id}/schemas", headers=headers, params=params
        )

    def get_subgraphs(
        self,
        assistant_id: str,
        namespace: str | None = None,
        recurse: bool = False,
        *,
        headers: Mapping[str, str] | None = None,
        params: QueryParamTypes | None = None,
    ) -> Subgraphs:
        """Get the schemas of an assistant by ID.

        Args:
            assistant_id: The ID of the assistant to get the schema of.
            headers: Optional custom headers to include with the request.
            params: Optional query parameters to include with the request.

        Returns:
            Subgraphs: The graph schema for the assistant.

        """  # noqa: E501
        get_params = {"recurse": recurse}
        if params:
            get_params = {**get_params, **params}
        if namespace is not None:
            return self.http.get(
                f"/assistants/{assistant_id}/subgraphs/{namespace}",
                params=get_params,
                headers=headers,
            )
        else:
            return self.http.get(
                f"/assistants/{assistant_id}/subgraphs",
                params=get_params,
                headers=headers,
            )

    def create(
        self,
        graph_id: str | None,
        config: Config | None = None,
        *,
        context: Context | None = None,
        metadata: Json = None,
        assistant_id: str | None = None,
        if_exists: OnConflictBehavior | None = None,
        name: str | None = None,
        headers: Mapping[str, str] | None = None,
        description: str | None = None,
        params: QueryParamTypes | None = None,
    ) -> Assistant:
        """Create a new assistant.

        Useful when graph is configurable and you want to create different assistants based on different configurations.

        Args:
            graph_id: The ID of the graph the assistant should use. The graph ID is normally set in your langgraph.json configuration.
            config: Configuration to use for the graph.
            context: Static context to add to the assistant.
                !!! version-added "Added in version 0.6.0"
            metadata: Metadata to add to assistant.
            assistant_id: Assistant ID to use, will default to a random UUID if not provided.
            if_exists: How to handle duplicate creation. Defaults to 'raise' under the hood.
                Must be either 'raise' (raise error if duplicate), or 'do_nothing' (return existing assistant).
            name: The name of the assistant. Defaults to 'Untitled' under the hood.
            headers: Optional custom headers to include with the request.
            description: Optional description of the assistant.
                The description field is available for langgraph-api server version>=0.0.45
            params: Optional query parameters to include with the request.

        Returns:
            The created assistant.

        ???+ example "Example Usage"

            ```python
            client = get_sync_client(url="http://localhost:2024")
            assistant = client.assistants.create(
                graph_id="agent",
                context={"model_name": "openai"},
                metadata={"number":1},
                assistant_id="my-assistant-id",
                if_exists="do_nothing",
                name="my_name"
            )
            ```
        """  # noqa: E501
        payload: dict[str, Any] = {
            "graph_id": graph_id,
        }
        if config:
            payload["config"] = config
        if context:
            payload["context"] = context
        if metadata:
            payload["metadata"] = metadata
        if assistant_id:
            payload["assistant_id"] = assistant_id
        if if_exists:
            payload["if_exists"] = if_exists
        if name:
            payload["name"] = name
        if description:
            payload["description"] = description
        return self.http.post(
            "/assistants", json=payload, headers=headers, params=params
        )

    def update(
        self,
        assistant_id: str,
        *,
        graph_id: str | None = None,
        config: Config | None = None,
        context: Context | None = None,
        metadata: Json = None,
        name: str | None = None,
        headers: Mapping[str, str] | None = None,
        description: str | None = None,
        params: QueryParamTypes | None = None,
    ) -> Assistant:
        """Update an assistant.

        Use this to point to a different graph, update the configuration, or change the metadata of an assistant.

        Args:
            assistant_id: Assistant to update.
            graph_id: The ID of the graph the assistant should use.
                The graph ID is normally set in your langgraph.json configuration. If `None`, assistant will keep pointing to same graph.
            config: Configuration to use for the graph.
            context: Static context to add to the assistant.
                !!! version-added "Added in version 0.6.0"
            metadata: Metadata to merge with existing assistant metadata.
            name: The new name for the assistant.
            headers: Optional custom headers to include with the request.
            description: Optional description of the assistant.
                The description field is available for langgraph-api server version>=0.0.45

        Returns:
            The updated assistant.

        ???+ example "Example Usage"

            ```python
            client = get_sync_client(url="http://localhost:2024")
            assistant = client.assistants.update(
                assistant_id='e280dad7-8618-443f-87f1-8e41841c180f',
                graph_id="other-graph",
                context={"model_name": "anthropic"},
                metadata={"number":2}
            )
            ```
        """  # noqa: E501
        payload: dict[str, Any] = {}
        if graph_id:
            payload["graph_id"] = graph_id
        if config:
            payload["config"] = config
        if context:
            payload["context"] = context
        if metadata:
            payload["metadata"] = metadata
        if name:
            payload["name"] = name
        if description:
            payload["description"] = description
        return self.http.patch(
            f"/assistants/{assistant_id}",
            json=payload,
            headers=headers,
            params=params,
        )

    def delete(
        self,
        assistant_id: str,
        *,
        headers: Mapping[str, str] | None = None,
        params: QueryParamTypes | None = None,
    ) -> None:
        """Delete an assistant.

        Args:
            assistant_id: The assistant ID to delete.
            headers: Optional custom headers to include with the request.
            params: Optional query parameters to include with the request.

        Returns:
            `None`

        ???+ example "Example Usage"

            ```python
            client = get_sync_client(url="http://localhost:2024")
            client.assistants.delete(
                assistant_id="my_assistant_id"
            )
            ```

        """  # noqa: E501
        self.http.delete(f"/assistants/{assistant_id}", headers=headers, params=params)

    def search(
        self,
        *,
        metadata: Json = None,
        graph_id: str | None = None,
        limit: int = 10,
        offset: int = 0,
        sort_by: AssistantSortBy | None = None,
        sort_order: SortOrder | None = None,
        select: list[AssistantSelectField] | None = None,
        headers: Mapping[str, str] | None = None,
        params: QueryParamTypes | None = None,
    ) -> list[Assistant]:
        """Search for assistants.

        Args:
            metadata: Metadata to filter by. Exact match filter for each KV pair.
            graph_id: The ID of the graph to filter by.
                The graph ID is normally set in your langgraph.json configuration.
            limit: The maximum number of results to return.
            offset: The number of results to skip.
            headers: Optional custom headers to include with the request.

        Returns:
            A list of assistants.

        ???+ example "Example Usage"

            ```python
            client = get_sync_client(url="http://localhost:2024")
            assistants = client.assistants.search(
                metadata = {"name":"my_name"},
                graph_id="my_graph_id",
                limit=5,
                offset=5
            )
            ```
        """
        payload: dict[str, Any] = {
            "limit": limit,
            "offset": offset,
        }
        if metadata:
            payload["metadata"] = metadata
        if graph_id:
            payload["graph_id"] = graph_id
        if sort_by:
            payload["sort_by"] = sort_by
        if sort_order:
            payload["sort_order"] = sort_order
        if select:
            payload["select"] = select
        return self.http.post(
            "/assistants/search",
            json=payload,
            headers=headers,
            params=params,
        )

    def count(
        self,
        *,
        metadata: Json = None,
        graph_id: str | None = None,
        headers: Mapping[str, str] | None = None,
        params: QueryParamTypes | None = None,
    ) -> int:
        """Count assistants matching filters.

        Args:
            metadata: Metadata to filter by. Exact match for each key/value.
            graph_id: Optional graph id to filter by.
            headers: Optional custom headers to include with the request.
            params: Optional query parameters to include with the request.

        Returns:
            int: Number of assistants matching the criteria.
        """
        payload: dict[str, Any] = {}
        if metadata:
            payload["metadata"] = metadata
        if graph_id:
            payload["graph_id"] = graph_id
        return self.http.post(
            "/assistants/count", json=payload, headers=headers, params=params
        )

    def get_versions(
        self,
        assistant_id: str,
        metadata: Json = None,
        limit: int = 10,
        offset: int = 0,
        *,
        headers: Mapping[str, str] | None = None,
        params: QueryParamTypes | None = None,
    ) -> list[AssistantVersion]:
        """List all versions of an assistant.

        Args:
            assistant_id: The assistant ID to get versions for.
            metadata: Metadata to filter versions by. Exact match filter for each KV pair.
            limit: The maximum number of versions to return.
            offset: The number of versions to skip.
            headers: Optional custom headers to include with the request.

        Returns:
            A list of assistants.

        ???+ example "Example Usage"

            ```python
            client = get_sync_client(url="http://localhost:2024")
            assistant_versions = client.assistants.get_versions(
                assistant_id="my_assistant_id"
            )
            ```

        """  # noqa: E501

        payload: dict[str, Any] = {
            "limit": limit,
            "offset": offset,
        }
        if metadata:
            payload["metadata"] = metadata
        return self.http.post(
            f"/assistants/{assistant_id}/versions",
            json=payload,
            headers=headers,
            params=params,
        )

    def set_latest(
        self,
        assistant_id: str,
        version: int,
        *,
        headers: Mapping[str, str] | None = None,
        params: QueryParamTypes | None = None,
    ) -> Assistant:
        """Change the version of an assistant.

        Args:
            assistant_id: The assistant ID to delete.
            version: The version to change to.
            headers: Optional custom headers to include with the request.

        Returns:
            `Assistant` Object.

        ???+ example "Example Usage"

            ```python
            client = get_sync_client(url="http://localhost:2024")
            new_version_assistant = client.assistants.set_latest(
                assistant_id="my_assistant_id",
                version=3
            )
            ```

        """  # noqa: E501

        payload: dict[str, Any] = {"version": version}

        return self.http.post(
            f"/assistants/{assistant_id}/latest",
            json=payload,
            headers=headers,
            params=params,
        )


class SyncThreadsClient:
    """Synchronous client for managing threads in LangGraph.

    This class provides methods to create, retrieve, and manage threads,
    which represent conversations or stateful interactions.

    ???+ example "Example"

        ```python
        client = get_sync_client(url="http://localhost:2024")
        thread = client.threads.create(metadata={"user_id": "123"})
        ```
    """

    def __init__(self, http: SyncHttpClient) -> None:
        self.http = http

    def get(
        self,
        thread_id: str,
        *,
        headers: Mapping[str, str] | None = None,
        params: QueryParamTypes | None = None,
    ) -> Thread:
        """Get a thread by ID.

        Args:
            thread_id: The ID of the thread to get.
            headers: Optional custom headers to include with the request.

        Returns:
            `Thread` object.

        ???+ example "Example Usage"

            ```python
            client = get_sync_client(url="http://localhost:2024")
            thread = client.threads.get(
                thread_id="my_thread_id"
            )
            print(thread)
            ```
            ```shell
            -----------------------------------------------------

            {
                'thread_id': 'my_thread_id',
                'created_at': '2024-07-18T18:35:15.540834+00:00',
                'updated_at': '2024-07-18T18:35:15.540834+00:00',
                'metadata': {'graph_id': 'agent'}
            }
            ```

        """  # noqa: E501

        return self.http.get(f"/threads/{thread_id}", headers=headers, params=params)

    def create(
        self,
        *,
        metadata: Json = None,
        thread_id: str | None = None,
        if_exists: OnConflictBehavior | None = None,
        supersteps: Sequence[dict[str, Sequence[dict[str, Any]]]] | None = None,
        graph_id: str | None = None,
        ttl: int | Mapping[str, Any] | None = None,
        headers: Mapping[str, str] | None = None,
        params: QueryParamTypes | None = None,
    ) -> Thread:
        """Create a new thread.

        Args:
            metadata: Metadata to add to thread.
            thread_id: ID of thread.
                If `None`, ID will be a randomly generated UUID.
            if_exists: How to handle duplicate creation. Defaults to 'raise' under the hood.
                Must be either 'raise' (raise error if duplicate), or 'do_nothing' (return existing thread).
            supersteps: Apply a list of supersteps when creating a thread, each containing a sequence of updates.
                Each update has `values` or `command` and `as_node`. Used for copying a thread between deployments.
            graph_id: Optional graph ID to associate with the thread.
            ttl: Optional time-to-live in minutes for the thread. You can pass an
                integer (minutes) or a mapping with keys `ttl` and optional
                `strategy` (defaults to "delete").
            headers: Optional custom headers to include with the request.

        Returns:
            The created `Thread`.

        ???+ example "Example Usage"

            ```python
            client = get_sync_client(url="http://localhost:2024")
            thread = client.threads.create(
                metadata={"number":1},
                thread_id="my-thread-id",
                if_exists="raise"
            )
            ```
            )
        """  # noqa: E501
        payload: dict[str, Any] = {}
        if thread_id:
            payload["thread_id"] = thread_id
        if metadata or graph_id:
            payload["metadata"] = {
                **(metadata or {}),
                **({"graph_id": graph_id} if graph_id else {}),
            }
        if if_exists:
            payload["if_exists"] = if_exists
        if supersteps:
            payload["supersteps"] = [
                {
                    "updates": [
                        {
                            "values": u["values"],
                            "command": u.get("command"),
                            "as_node": u["as_node"],
                        }
                        for u in s["updates"]
                    ]
                }
                for s in supersteps
            ]
        if ttl is not None:
            if isinstance(ttl, (int, float)):
                payload["ttl"] = {"ttl": ttl, "strategy": "delete"}
            else:
                payload["ttl"] = ttl

        return self.http.post("/threads", json=payload, headers=headers, params=params)

    def update(
        self,
        thread_id: str,
        *,
        metadata: Mapping[str, Any],
        ttl: int | Mapping[str, Any] | None = None,
        headers: Mapping[str, str] | None = None,
        params: QueryParamTypes | None = None,
    ) -> Thread:
        """Update a thread.

        Args:
            thread_id: ID of thread to update.
            metadata: Metadata to merge with existing thread metadata.
            ttl: Optional time-to-live in minutes for the thread. You can pass an
                integer (minutes) or a mapping with keys `ttl` and optional
                `strategy` (defaults to "delete").
            headers: Optional custom headers to include with the request.
            params: Optional query parameters to include with the request.

        Returns:
            The created `Thread`.

        ???+ example "Example Usage"

            ```python
            client = get_sync_client(url="http://localhost:2024")
            thread = client.threads.update(
                thread_id="my-thread-id",
                metadata={"number":1},
                ttl=43_200,
            )
            ```
        """  # noqa: E501
        payload: dict[str, Any] = {"metadata": metadata}
        if ttl is not None:
            if isinstance(ttl, (int, float)):
                payload["ttl"] = {"ttl": ttl, "strategy": "delete"}
            else:
                payload["ttl"] = ttl
        return self.http.patch(
            f"/threads/{thread_id}",
            json=payload,
            headers=headers,
            params=params,
        )

    def delete(
        self,
        thread_id: str,
        *,
        headers: Mapping[str, str] | None = None,
        params: QueryParamTypes | None = None,
    ) -> None:
        """Delete a thread.

        Args:
            thread_id: The ID of the thread to delete.
            headers: Optional custom headers to include with the request.
            params: Optional query parameters to include with the request.

        Returns:
            `None`

        ???+ example "Example Usage"

            ```python
            client.threads.delete(
                thread_id="my_thread_id"
            )
            ```

        """  # noqa: E501
        self.http.delete(f"/threads/{thread_id}", headers=headers, params=params)

    def search(
        self,
        *,
        metadata: Json = None,
        values: Json = None,
        ids: Sequence[str] | None = None,
        status: ThreadStatus | None = None,
        limit: int = 10,
        offset: int = 0,
        sort_by: ThreadSortBy | None = None,
        sort_order: SortOrder | None = None,
        select: list[ThreadSelectField] | None = None,
        headers: Mapping[str, str] | None = None,
        params: QueryParamTypes | None = None,
    ) -> list[Thread]:
        """Search for threads.

        Args:
            metadata: Thread metadata to filter on.
            values: State values to filter on.
            ids: List of thread IDs to filter by.
            status: Thread status to filter on.
                Must be one of 'idle', 'busy', 'interrupted' or 'error'.
            limit: Limit on number of threads to return.
            offset: Offset in threads table to start search from.
            headers: Optional custom headers to include with the request.

        Returns:
            List of the threads matching the search parameters.

        ???+ example "Example Usage"

            ```python
            client = get_sync_client(url="http://localhost:2024")
            threads = client.threads.search(
                metadata={"number":1},
                status="interrupted",
                limit=15,
                offset=5
            )
            ```
        """  # noqa: E501
        payload: dict[str, Any] = {
            "limit": limit,
            "offset": offset,
        }
        if metadata:
            payload["metadata"] = metadata
        if values:
            payload["values"] = values
        if ids:
            payload["ids"] = ids
        if status:
            payload["status"] = status
        if sort_by:
            payload["sort_by"] = sort_by
        if sort_order:
            payload["sort_order"] = sort_order
        if select:
            payload["select"] = select
        return self.http.post(
            "/threads/search", json=payload, headers=headers, params=params
        )

    def count(
        self,
        *,
        metadata: Json = None,
        values: Json = None,
        status: ThreadStatus | None = None,
        headers: Mapping[str, str] | None = None,
        params: QueryParamTypes | None = None,
    ) -> int:
        """Count threads matching filters.

        Args:
            metadata: Thread metadata to filter on.
            values: State values to filter on.
            status: Thread status to filter on.
            headers: Optional custom headers to include with the request.
            params: Optional query parameters to include with the request.

        Returns:
            int: Number of threads matching the criteria.
        """
        payload: dict[str, Any] = {}
        if metadata:
            payload["metadata"] = metadata
        if values:
            payload["values"] = values
        if status:
            payload["status"] = status
        return self.http.post(
            "/threads/count", json=payload, headers=headers, params=params
        )

    def copy(
        self,
        thread_id: str,
        *,
        headers: Mapping[str, str] | None = None,
        params: QueryParamTypes | None = None,
    ) -> None:
        """Copy a thread.

        Args:
            thread_id: The ID of the thread to copy.
            headers: Optional custom headers to include with the request.
            params: Optional query parameters to include with the request.

        Returns:
            `None`

        ???+ example "Example Usage"

            ```python
            client = get_sync_client(url="http://localhost:2024")
            client.threads.copy(
                thread_id="my_thread_id"
            )
            ```

        """  # noqa: E501
        return self.http.post(
            f"/threads/{thread_id}/copy", json=None, headers=headers, params=params
        )

    def get_state(
        self,
        thread_id: str,
        checkpoint: Checkpoint | None = None,
        checkpoint_id: str | None = None,  # deprecated
        *,
        subgraphs: bool = False,
        headers: Mapping[str, str] | None = None,
        params: QueryParamTypes | None = None,
    ) -> ThreadState:
        """Get the state of a thread.

        Args:
            thread_id: The ID of the thread to get the state of.
            checkpoint: The checkpoint to get the state of.
            subgraphs: Include subgraphs states.
            headers: Optional custom headers to include with the request.

        Returns:
            The thread of the state.

        ???+ example "Example Usage"

            ```python
            client = get_sync_client(url="http://localhost:2024")
            thread_state = client.threads.get_state(
                thread_id="my_thread_id",
                checkpoint_id="my_checkpoint_id"
            )
            print(thread_state)
            ```

            ```shell
            ----------------------------------------------------------------------------------------------------------------------------------------------------------------------

            {
                'values': {
                    'messages': [
                        {
                            'content': 'how are you?',
                            'additional_kwargs': {},
                            'response_metadata': {},
                            'type': 'human',
                            'name': None,
                            'id': 'fe0a5778-cfe9-42ee-b807-0adaa1873c10',
                            'example': False
                        },
                        {
                            'content': "I'm doing well, thanks for asking! I'm an AI assistant created by Anthropic to be helpful, honest, and harmless.",
                            'additional_kwargs': {},
                            'response_metadata': {},
                            'type': 'ai',
                            'name': None,
                            'id': 'run-159b782c-b679-4830-83c6-cef87798fe8b',
                            'example': False,
                            'tool_calls': [],
                            'invalid_tool_calls': [],
                            'usage_metadata': None
                        }
                    ]
                },
                'next': [],
                'checkpoint':
                    {
                        'thread_id': 'e2496803-ecd5-4e0c-a779-3226296181c2',
                        'checkpoint_ns': '',
                        'checkpoint_id': '1ef4a9b8-e6fb-67b1-8001-abd5184439d1'
                    }
                'metadata':
                    {
                        'step': 1,
                        'run_id': '1ef4a9b8-d7da-679a-a45a-872054341df2',
                        'source': 'loop',
                        'writes':
                            {
                                'agent':
                                    {
                                        'messages': [
                                            {
                                                'id': 'run-159b782c-b679-4830-83c6-cef87798fe8b',
                                                'name': None,
                                                'type': 'ai',
                                                'content': "I'm doing well, thanks for asking! I'm an AI assistant created by Anthropic to be helpful, honest, and harmless.",
                                                'example': False,
                                                'tool_calls': [],
                                                'usage_metadata': None,
                                                'additional_kwargs': {},
                                                'response_metadata': {},
                                                'invalid_tool_calls': []
                                            }
                                        ]
                                    }
                            },
                'user_id': None,
                'graph_id': 'agent',
                'thread_id': 'e2496803-ecd5-4e0c-a779-3226296181c2',
                'created_by': 'system',
                'assistant_id': 'fe096781-5601-53d2-b2f6-0d3403f7e9ca'},
                'created_at': '2024-07-25T15:35:44.184703+00:00',
                'parent_config':
                    {
                        'thread_id': 'e2496803-ecd5-4e0c-a779-3226296181c2',
                        'checkpoint_ns': '',
                        'checkpoint_id': '1ef4a9b8-d80d-6fa7-8000-9300467fad0f'
                    }
            }
            ```

        """  # noqa: E501
        if checkpoint:
            return self.http.post(
                f"/threads/{thread_id}/state/checkpoint",
                json={"checkpoint": checkpoint, "subgraphs": subgraphs},
                headers=headers,
                params=params,
            )
        elif checkpoint_id:
            get_params = {"subgraphs": subgraphs}
            if params:
                get_params = {**get_params, **params}
            return self.http.get(
                f"/threads/{thread_id}/state/{checkpoint_id}",
                params=get_params,
                headers=headers,
            )
        else:
            get_params = {"subgraphs": subgraphs}
            if params:
                get_params = {**get_params, **params}
            return self.http.get(
                f"/threads/{thread_id}/state",
                params=get_params,
                headers=headers,
            )

    def update_state(
        self,
        thread_id: str,
        values: dict[str, Any] | Sequence[dict] | None,
        *,
        as_node: str | None = None,
        checkpoint: Checkpoint | None = None,
        checkpoint_id: str | None = None,  # deprecated
        headers: Mapping[str, str] | None = None,
        params: QueryParamTypes | None = None,
    ) -> ThreadUpdateStateResponse:
        """Update the state of a thread.

        Args:
            thread_id: The ID of the thread to update.
            values: The values to update the state with.
            as_node: Update the state as if this node had just executed.
            checkpoint: The checkpoint to update the state of.
            headers: Optional custom headers to include with the request.

        Returns:
            Response after updating a thread's state.

        ???+ example "Example Usage"

            ```python

            response = await client.threads.update_state(
                thread_id="my_thread_id",
                values={"messages":[{"role": "user", "content": "hello!"}]},
                as_node="my_node",
            )
            print(response)

            ----------------------------------------------------------------------------------------------------------------------------------------------------------------------

            {
                'checkpoint': {
                    'thread_id': 'e2496803-ecd5-4e0c-a779-3226296181c2',
                    'checkpoint_ns': '',
                    'checkpoint_id': '1ef4a9b8-e6fb-67b1-8001-abd5184439d1',
                    'checkpoint_map': {}
                }
            }
            ```

        """  # noqa: E501
        payload: dict[str, Any] = {
            "values": values,
        }
        if checkpoint_id:
            payload["checkpoint_id"] = checkpoint_id
        if checkpoint:
            payload["checkpoint"] = checkpoint
        if as_node:
            payload["as_node"] = as_node
        return self.http.post(
            f"/threads/{thread_id}/state", json=payload, headers=headers, params=params
        )

    def get_history(
        self,
        thread_id: str,
        *,
        limit: int = 10,
        before: str | Checkpoint | None = None,
        metadata: Mapping[str, Any] | None = None,
        checkpoint: Checkpoint | None = None,
        headers: Mapping[str, str] | None = None,
        params: QueryParamTypes | None = None,
    ) -> list[ThreadState]:
        """Get the state history of a thread.

        Args:
            thread_id: The ID of the thread to get the state history for.
            checkpoint: Return states for this subgraph. If empty defaults to root.
            limit: The maximum number of states to return.
            before: Return states before this checkpoint.
            metadata: Filter states by metadata key-value pairs.
            headers: Optional custom headers to include with the request.

        Returns:
            The state history of the `Thread`.

        ???+ example "Example Usage"

            ```python

            thread_state = client.threads.get_history(
                thread_id="my_thread_id",
                limit=5,
                before="my_timestamp",
                metadata={"name":"my_name"}
            )
            ```

        """  # noqa: E501
        payload: dict[str, Any] = {
            "limit": limit,
        }
        if before:
            payload["before"] = before
        if metadata:
            payload["metadata"] = metadata
        if checkpoint:
            payload["checkpoint"] = checkpoint
        return self.http.post(
            f"/threads/{thread_id}/history",
            json=payload,
            headers=headers,
            params=params,
        )

    def join_stream(
        self,
        thread_id: str,
        *,
        stream_mode: ThreadStreamMode | Sequence[ThreadStreamMode] = "run_modes",
        last_event_id: str | None = None,
        headers: Mapping[str, str] | None = None,
        params: QueryParamTypes | None = None,
    ) -> Iterator[StreamPart]:
        """Get a stream of events for a thread.

        Args:
            thread_id: The ID of the thread to get the stream for.
            last_event_id: The ID of the last event to get.
            headers: Optional custom headers to include with the request.
            params: Optional query parameters to include with the request.

        Returns:
            An iterator of stream parts.

        ???+ example "Example Usage"

            ```python

            for chunk in client.threads.join_stream(
                thread_id="my_thread_id",
                last_event_id="my_event_id",
                stream_mode="run_modes",
            ):
                print(chunk)
            ```

        """  # noqa: E501
        query_params = {
            "stream_mode": stream_mode,
        }
        if params:
            query_params.update(params)
        return self.http.stream(
            f"/threads/{thread_id}/stream",
            "GET",
            headers={
                **({"Last-Event-ID": last_event_id} if last_event_id else {}),
                **(headers or {}),
            },
            params=query_params,
        )


class SyncRunsClient:
    """Synchronous client for managing runs in LangGraph.

    This class provides methods to create, retrieve, and manage runs, which represent
    individual executions of graphs.

    ???+ example "Example"

        ```python
        client = get_sync_client(url="http://localhost:2024")
        run = client.runs.create(thread_id="thread_123", assistant_id="asst_456")
        ```
    """

    def __init__(self, http: SyncHttpClient) -> None:
        self.http = http

    @overload
    def stream(
        self,
        thread_id: str,
        assistant_id: str,
        *,
        input: Mapping[str, Any] | None = None,
        command: Command | None = None,
        stream_mode: StreamMode | Sequence[StreamMode] = "values",
        stream_subgraphs: bool = False,
        metadata: Mapping[str, Any] | None = None,
        config: Config | None = None,
        context: Context | None = None,
        checkpoint: Checkpoint | None = None,
        checkpoint_id: str | None = None,
        checkpoint_during: bool | None = None,
        interrupt_before: All | Sequence[str] | None = None,
        interrupt_after: All | Sequence[str] | None = None,
        feedback_keys: Sequence[str] | None = None,
        on_disconnect: DisconnectMode | None = None,
        webhook: str | None = None,
        multitask_strategy: MultitaskStrategy | None = None,
        if_not_exists: IfNotExists | None = None,
        after_seconds: int | None = None,
        headers: Mapping[str, str] | None = None,
        params: QueryParamTypes | None = None,
        on_run_created: Callable[[RunCreateMetadata], None] | None = None,
    ) -> Iterator[StreamPart]: ...

    @overload
    def stream(
        self,
        thread_id: None,
        assistant_id: str,
        *,
        input: Mapping[str, Any] | None = None,
        command: Command | None = None,
        stream_mode: StreamMode | Sequence[StreamMode] = "values",
        stream_subgraphs: bool = False,
        stream_resumable: bool = False,
        metadata: Mapping[str, Any] | None = None,
        config: Config | None = None,
        context: Context | None = None,
        checkpoint_during: bool | None = None,
        interrupt_before: All | Sequence[str] | None = None,
        interrupt_after: All | Sequence[str] | None = None,
        feedback_keys: Sequence[str] | None = None,
        on_disconnect: DisconnectMode | None = None,
        on_completion: OnCompletionBehavior | None = None,
        if_not_exists: IfNotExists | None = None,
        webhook: str | None = None,
        after_seconds: int | None = None,
        headers: Mapping[str, str] | None = None,
        params: QueryParamTypes | None = None,
        on_run_created: Callable[[RunCreateMetadata], None] | None = None,
    ) -> Iterator[StreamPart]: ...

    def stream(
        self,
        thread_id: str | None,
        assistant_id: str,
        *,
        input: Mapping[str, Any] | None = None,
        command: Command | None = None,
        stream_mode: StreamMode | Sequence[StreamMode] = "values",
        stream_subgraphs: bool = False,
        stream_resumable: bool = False,
        metadata: Mapping[str, Any] | None = None,
        config: Config | None = None,
        context: Context | None = None,
        checkpoint: Checkpoint | None = None,
        checkpoint_id: str | None = None,
        checkpoint_during: bool | None = None,  # deprecated
        interrupt_before: All | Sequence[str] | None = None,
        interrupt_after: All | Sequence[str] | None = None,
        feedback_keys: Sequence[str] | None = None,
        on_disconnect: DisconnectMode | None = None,
        on_completion: OnCompletionBehavior | None = None,
        webhook: str | None = None,
        multitask_strategy: MultitaskStrategy | None = None,
        if_not_exists: IfNotExists | None = None,
        after_seconds: int | None = None,
        headers: Mapping[str, str] | None = None,
        params: QueryParamTypes | None = None,
        on_run_created: Callable[[RunCreateMetadata], None] | None = None,
        durability: Durability | None = None,
    ) -> Iterator[StreamPart]:
        """Create a run and stream the results.

        Args:
            thread_id: the thread ID to assign to the thread.
                If `None` will create a stateless run.
            assistant_id: The assistant ID or graph name to stream from.
                If using graph name, will default to first assistant created from that graph.
            input: The input to the graph.
            command: The command to execute.
            stream_mode: The stream mode(s) to use.
            stream_subgraphs: Whether to stream output from subgraphs.
            stream_resumable: Whether the stream is considered resumable.
                If true, the stream can be resumed and replayed in its entirety even after disconnection.
            metadata: Metadata to assign to the run.
            config: The configuration for the assistant.
            context: Static context to add to the assistant.
                !!! version-added "Added in version 0.6.0"
            checkpoint: The checkpoint to resume from.
            checkpoint_during: (deprecated) Whether to checkpoint during the run (or only at the end/interruption).
            interrupt_before: Nodes to interrupt immediately before they get executed.
            interrupt_after: Nodes to Nodes to interrupt immediately after they get executed.
            feedback_keys: Feedback keys to assign to run.
            on_disconnect: The disconnect mode to use.
                Must be one of 'cancel' or 'continue'.
            on_completion: Whether to delete or keep the thread created for a stateless run.
                Must be one of 'delete' or 'keep'.
            webhook: Webhook to call after LangGraph API call is done.
            multitask_strategy: Multitask strategy to use.
                Must be one of 'reject', 'interrupt', 'rollback', or 'enqueue'.
            if_not_exists: How to handle missing thread. Defaults to 'reject'.
                Must be either 'reject' (raise error if missing), or 'create' (create new thread).
            after_seconds: The number of seconds to wait before starting the run.
                Use to schedule future runs.
            headers: Optional custom headers to include with the request.
            on_run_created: Optional callback to call when a run is created.
            durability: The durability to use for the run. Values are "sync", "async", or "exit".
                "async" means checkpoints are persisted async while next graph step executes, replaces checkpoint_during=True
                "sync" means checkpoints are persisted sync after graph step executes, replaces checkpoint_during=False
                "exit" means checkpoints are only persisted when the run exits, does not save intermediate steps


        Returns:
            Iterator of stream results.

        ???+ example "Example Usage"

            ```python
            client = get_sync_client(url="http://localhost:2024")
            async for chunk in client.runs.stream(
                thread_id=None,
                assistant_id="agent",
                input={"messages": [{"role": "user", "content": "how are you?"}]},
                stream_mode=["values","debug"],
                metadata={"name":"my_run"},
                context={"model_name": "anthropic"},
                interrupt_before=["node_to_stop_before_1","node_to_stop_before_2"],
                interrupt_after=["node_to_stop_after_1","node_to_stop_after_2"],
                feedback_keys=["my_feedback_key_1","my_feedback_key_2"],
                webhook="https://my.fake.webhook.com",
                multitask_strategy="interrupt"
            ):
                print(chunk)
            ```
            ```shell
            ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

            StreamPart(event='metadata', data={'run_id': '1ef4a9b8-d7da-679a-a45a-872054341df2'})
            StreamPart(event='values', data={'messages': [{'content': 'how are you?', 'additional_kwargs': {}, 'response_metadata': {}, 'type': 'human', 'name': None, 'id': 'fe0a5778-cfe9-42ee-b807-0adaa1873c10', 'example': False}]})
            StreamPart(event='values', data={'messages': [{'content': 'how are you?', 'additional_kwargs': {}, 'response_metadata': {}, 'type': 'human', 'name': None, 'id': 'fe0a5778-cfe9-42ee-b807-0adaa1873c10', 'example': False}, {'content': "I'm doing well, thanks for asking! I'm an AI assistant created by Anthropic to be helpful, honest, and harmless.", 'additional_kwargs': {}, 'response_metadata': {}, 'type': 'ai', 'name': None, 'id': 'run-159b782c-b679-4830-83c6-cef87798fe8b', 'example': False, 'tool_calls': [], 'invalid_tool_calls': [], 'usage_metadata': None}]})
            StreamPart(event='end', data=None)
            ```
        """  # noqa: E501
        if checkpoint_during is not None:
            warnings.warn(
                "`checkpoint_during` is deprecated and will be removed in a future version. Use `durability` instead.",
                DeprecationWarning,
                stacklevel=2,
            )
        payload = {
            "input": input,
            "command": (
                {k: v for k, v in command.items() if v is not None} if command else None
            ),
            "config": config,
            "context": context,
            "metadata": metadata,
            "stream_mode": stream_mode,
            "stream_subgraphs": stream_subgraphs,
            "stream_resumable": stream_resumable,
            "assistant_id": assistant_id,
            "interrupt_before": interrupt_before,
            "interrupt_after": interrupt_after,
            "feedback_keys": feedback_keys,
            "webhook": webhook,
            "checkpoint": checkpoint,
            "checkpoint_id": checkpoint_id,
            "checkpoint_during": checkpoint_during,
            "multitask_strategy": multitask_strategy,
            "if_not_exists": if_not_exists,
            "on_disconnect": on_disconnect,
            "on_completion": on_completion,
            "after_seconds": after_seconds,
            "durability": durability,
        }
        endpoint = (
            f"/threads/{thread_id}/runs/stream"
            if thread_id is not None
            else "/runs/stream"
        )

        def on_response(res: httpx.Response):
            """Callback function to handle the response."""
            if on_run_created and (metadata := _get_run_metadata_from_response(res)):
                on_run_created(metadata)

        return self.http.stream(
            endpoint,
            "POST",
            json={k: v for k, v in payload.items() if v is not None},
            params=params,
            headers=headers,
            on_response=on_response if on_run_created else None,
        )

    @overload
    def create(
        self,
        thread_id: None,
        assistant_id: str,
        *,
        input: Mapping[str, Any] | None = None,
        command: Command | None = None,
        stream_mode: StreamMode | Sequence[StreamMode] = "values",
        stream_subgraphs: bool = False,
        stream_resumable: bool = False,
        metadata: Mapping[str, Any] | None = None,
        config: Config | None = None,
        context: Context | None = None,
        checkpoint_during: bool | None = None,
        interrupt_before: All | Sequence[str] | None = None,
        interrupt_after: All | Sequence[str] | None = None,
        webhook: str | None = None,
        on_completion: OnCompletionBehavior | None = None,
        if_not_exists: IfNotExists | None = None,
        after_seconds: int | None = None,
        headers: Mapping[str, str] | None = None,
        params: QueryParamTypes | None = None,
        on_run_created: Callable[[RunCreateMetadata], None] | None = None,
    ) -> Run: ...

    @overload
    def create(
        self,
        thread_id: str,
        assistant_id: str,
        *,
        input: Mapping[str, Any] | None = None,
        command: Command | None = None,
        stream_mode: StreamMode | Sequence[StreamMode] = "values",
        stream_subgraphs: bool = False,
        stream_resumable: bool = False,
        metadata: Mapping[str, Any] | None = None,
        config: Config | None = None,
        context: Context | None = None,
        checkpoint: Checkpoint | None = None,
        checkpoint_id: str | None = None,
        checkpoint_during: bool | None = None,
        interrupt_before: All | Sequence[str] | None = None,
        interrupt_after: All | Sequence[str] | None = None,
        webhook: str | None = None,
        multitask_strategy: MultitaskStrategy | None = None,
        if_not_exists: IfNotExists | None = None,
        after_seconds: int | None = None,
        headers: Mapping[str, str] | None = None,
        params: QueryParamTypes | None = None,
        on_run_created: Callable[[RunCreateMetadata], None] | None = None,
    ) -> Run: ...

    def create(
        self,
        thread_id: str | None,
        assistant_id: str,
        *,
        input: Mapping[str, Any] | None = None,
        command: Command | None = None,
        stream_mode: StreamMode | Sequence[StreamMode] = "values",
        stream_subgraphs: bool = False,
        stream_resumable: bool = False,
        metadata: Mapping[str, Any] | None = None,
        config: Config | None = None,
        context: Context | None = None,
        checkpoint: Checkpoint | None = None,
        checkpoint_id: str | None = None,
        checkpoint_during: bool | None = None,  # deprecated
        i
```
> [truncated]

---
https://github.com/langchain-ai/langgraph/blob/main/libs/sdk-py/langgraph_sdk/errors.py
```py
from __future__ import annotations

import logging
import sys
from typing import Any, Literal, cast

import httpx
import orjson

logger = logging.getLogger(__name__)


class LangGraphError(Exception):
    pass


class APIError(httpx.HTTPStatusError, LangGraphError):
    message: str
    request: httpx.Request

    body: object | None
    code: str | None
    param: str | None
    type: str | None

    def __init__(
        self, message: str, response: httpx.Response, *, body: object | None
    ) -> None:
        httpx.HTTPStatusError.__init__(
            self, message, request=response.request, response=response
        )
        LangGraphError.__init__(self)

        self.request = response.request
        self.message = message
        self.body = body

        if isinstance(body, dict):
            b = cast(dict[str, Any], body)
            # Best-effort extraction of common fields if present
            code_val = b.get("code")
            self.code = code_val if isinstance(code_val, str) else None
            param_val = b.get("param")
            self.param = param_val if isinstance(param_val, str) else None
            t = b.get("type")
            self.type = t if isinstance(t, str) else None
        else:
            self.code = None
            self.param = None
            self.type = None


class APIResponseValidationError(APIError):
    response: httpx.Response
    status_code: int

    def __init__(
        self,
        response: httpx.Response,
        body: object | None,
        *,
        message: str | None = None,
    ) -> None:
        super().__init__(
            message or "Data returned by API invalid for expected schema.",
            response,
            body=body,
        )
        self.response = response
        self.status_code = response.status_code


class APIStatusError(APIError):
    response: httpx.Response
    status_code: int
    request_id: str | None

    def __init__(
        self, message: str, *, response: httpx.Response, body: object | None
    ) -> None:
        super().__init__(message, response, body=body)
        self.response = response
        self.status_code = response.status_code
        self.request_id = response.headers.get("x-request-id")


class APIConnectionError(APIError):
    def __init__(
        self, *, message: str = "Connection error.", request: httpx.Request
    ) -> None:
        super().__init__(message, request, body=None)


class APITimeoutError(APIConnectionError):
    def __init__(self, request: httpx.Request) -> None:
        super().__init__(message="Request timed out.", request=request)


class BadRequestError(APIStatusError):
    status_code: Literal[400] = 400


class AuthenticationError(APIStatusError):
    status_code: Literal[401] = 401


class PermissionDeniedError(APIStatusError):
    status_code: Literal[403] = 403


class NotFoundError(APIStatusError):
    status_code: Literal[404] = 404


class ConflictError(APIStatusError):
    status_code: Literal[409] = 409


class UnprocessableEntityError(APIStatusError):
    status_code: Literal[422] = 422


class RateLimitError(APIStatusError):
    status_code: Literal[429] = 429


class InternalServerError(APIStatusError):
    pass


def _extract_error_message(body: object | None, fallback: str) -> str:
    if isinstance(body, dict):
        b = cast(dict[str, Any], body)
        for key in ("message", "detail", "error"):
            val = b.get(key)
            if isinstance(val, str) and val:
                return val
        # Sometimes errors are structured like {"error": {"message": "..."}}
        err = b.get("error")
        if isinstance(err, dict):
            e = cast(dict[str, Any], err)
            for key in ("message", "detail"):
                val = e.get(key)
                if isinstance(val, str) and val:
                    return val
    return fallback


async def _adecode_error_body(r: httpx.Response) -> object | None:
    try:
        data = await r.aread()
    except Exception:
        return None
    if not data:
        return None
    try:
        return orjson.loads(data)
    except Exception:
        try:
            return data.decode()
        except Exception:
            return None


def _decode_error_body(r: httpx.Response) -> object | None:
    try:
        data = r.read()
    except Exception:
        return None
    if not data:
        return None
    try:
        return orjson.loads(data)
    except Exception:
        try:
            return data.decode()
        except Exception:
            return None


def _map_status_error(response: httpx.Response, body: object | None) -> APIStatusError:
    status = response.status_code
    reason = response.reason_phrase or "HTTP Error"
    message = _extract_error_message(body, f"{status} {reason}")
    if status == 400:
        return BadRequestError(message, response=response, body=body)
    if status == 401:
        return AuthenticationError(message, response=response, body=body)
    if status == 403:
        return PermissionDeniedError(message, response=response, body=body)
    if status == 404:
        return NotFoundError(message, response=response, body=body)
    if status == 409:
        return ConflictError(message, response=response, body=body)
    if status == 422:
        return UnprocessableEntityError(message, response=response, body=body)
    if status == 429:
        return RateLimitError(message, response=response, body=body)
    if 500 <= status:
        return InternalServerError(message, response=response, body=body)
    return APIStatusError(message, response=response, body=body)


async def _araise_for_status_typed(r: httpx.Response) -> None:
    if r.status_code < 400:
        return
    body = await _adecode_error_body(r)
    err = _map_status_error(r, body)
    # Log for older Python versions without Exception notes
    if not (sys.version_info >= (3, 11)):
        logger.error(f"Error from langgraph-api: {getattr(err, 'message', '')}")
    raise err


def _raise_for_status_typed(r: httpx.Response) -> None:
    if r.status_code < 400:
        return
    body = _decode_error_body(r)
    err = _map_status_error(r, body)
    if not (sys.version_info >= (3, 11)):
        logger.error(f"Error from langgraph-api: {getattr(err, 'message', '')}")
    raise err

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/sdk-py/langgraph_sdk/py.typed
```typed

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/sdk-py/langgraph_sdk/schema.py
```py
"""Data models for interacting with the LangGraph API."""

from __future__ import annotations

from collections.abc import Mapping, Sequence
from datetime import datetime
from typing import (
    Any,
    Literal,
    NamedTuple,
    TypeAlias,
    TypedDict,
)

Json = dict[str, Any] | None
"""Represents a JSON-like structure, which can be None or a dictionary with string keys and any values."""

RunStatus = Literal["pending", "running", "error", "success", "timeout", "interrupted"]
"""
Represents the status of a run:
- "pending": The run is waiting to start.
- "running": The run is currently executing.
- "error": The run encountered an error and stopped.
- "success": The run completed successfully.
- "timeout": The run exceeded its time limit.
- "interrupted": The run was manually stopped or interrupted.
"""

ThreadStatus = Literal["idle", "busy", "interrupted", "error"]
"""
Represents the status of a thread:
- "idle": The thread is not currently processing any task.
- "busy": The thread is actively processing a task.
- "interrupted": The thread's execution was interrupted.
- "error": An exception occurred during task processing.
"""

ThreadStreamMode = Literal["run_modes", "lifecycle", "state_update"]
"""
Defines the mode of streaming:
- "run_modes": Stream the same events as the runs on thread, as well as run_done events.
- "lifecycle": Stream only run start/end events.
- "state_update": Stream state updates on the thread.
"""

StreamMode = Literal[
    "values",
    "messages",
    "updates",
    "events",
    "tasks",
    "checkpoints",
    "debug",
    "custom",
    "messages-tuple",
]
"""
Defines the mode of streaming:
- "values": Stream only the values.
- "messages": Stream complete messages.
- "updates": Stream updates to the state.
- "events": Stream events occurring during execution.
- "checkpoints": Stream checkpoints as they are created.
- "tasks": Stream task start and finish events.
- "debug": Stream detailed debug information.
- "custom": Stream custom events.
"""

DisconnectMode = Literal["cancel", "continue"]
"""
Specifies behavior on disconnection:
- "cancel": Cancel the operation on disconnection.
- "continue": Continue the operation even if disconnected.
"""

MultitaskStrategy = Literal["reject", "interrupt", "rollback", "enqueue"]
"""
Defines how to handle multiple tasks:
- "reject": Reject new tasks when busy.
- "interrupt": Interrupt current task for new ones.
- "rollback": Roll back current task and start new one.
- "enqueue": Queue new tasks for later execution.
"""

OnConflictBehavior = Literal["raise", "do_nothing"]
"""
Specifies behavior on conflict:
- "raise": Raise an exception when a conflict occurs.
- "do_nothing": Ignore conflicts and proceed.
"""

OnCompletionBehavior = Literal["delete", "keep"]
"""
Defines action after completion:
- "delete": Delete resources after completion.
- "keep": Retain resources after completion.
"""

Durability = Literal["sync", "async", "exit"]
"""Durability mode for the graph execution.
- `"sync"`: Changes are persisted synchronously before the next step starts.
- `"async"`: Changes are persisted asynchronously while the next step executes.
- `"exit"`: Changes are persisted only when the graph exits."""

All = Literal["*"]
"""Represents a wildcard or 'all' selector."""

IfNotExists = Literal["create", "reject"]
"""
Specifies behavior if the thread doesn't exist:
- "create": Create a new thread if it doesn't exist.
- "reject": Reject the operation if the thread doesn't exist.
"""

CancelAction = Literal["interrupt", "rollback"]
"""
Action to take when cancelling the run.
- "interrupt": Simply cancel the run.
- "rollback": Cancel the run. Then delete the run and associated checkpoints.
"""

AssistantSortBy = Literal[
    "assistant_id", "graph_id", "name", "created_at", "updated_at"
]
"""
The field to sort by.
"""

ThreadSortBy = Literal["thread_id", "status", "created_at", "updated_at"]
"""
The field to sort by.
"""

CronSortBy = Literal[
    "cron_id", "assistant_id", "thread_id", "created_at", "updated_at", "next_run_date"
]
"""
The field to sort by.
"""

SortOrder = Literal["asc", "desc"]
"""
The order to sort by.
"""

Context: TypeAlias = dict[str, Any]


class Config(TypedDict, total=False):
    """Configuration options for a call."""

    tags: list[str]
    """
    Tags for this call and any sub-calls (eg. a Chain calling an LLM).
    You can use these to filter calls.
    """

    recursion_limit: int
    """
    Maximum number of times a call can recurse. If not provided, defaults to 25.
    """

    configurable: dict[str, Any]
    """
    Runtime values for attributes previously made configurable on this Runnable,
    or sub-Runnables, through .configurable_fields() or .configurable_alternatives().
    Check .output_schema() for a description of the attributes that have been made 
    configurable.
    """


class Checkpoint(TypedDict):
    """Represents a checkpoint in the execution process."""

    thread_id: str
    """Unique identifier for the thread associated with this checkpoint."""
    checkpoint_ns: str
    """Namespace for the checkpoint; used internally to manage subgraph state."""
    checkpoint_id: str | None
    """Optional unique identifier for the checkpoint itself."""
    checkpoint_map: dict[str, Any] | None
    """Optional dictionary containing checkpoint-specific data."""


class GraphSchema(TypedDict):
    """Defines the structure and properties of a graph."""

    graph_id: str
    """The ID of the graph."""
    input_schema: dict | None
    """The schema for the graph input.
    Missing if unable to generate JSON schema from graph."""
    output_schema: dict | None
    """The schema for the graph output.
    Missing if unable to generate JSON schema from graph."""
    state_schema: dict | None
    """The schema for the graph state.
    Missing if unable to generate JSON schema from graph."""
    config_schema: dict | None
    """The schema for the graph config.
    Missing if unable to generate JSON schema from graph."""
    context_schema: dict | None
    """The schema for the graph context.
    Missing if unable to generate JSON schema from graph."""


Subgraphs = dict[str, GraphSchema]


class AssistantBase(TypedDict):
    """Base model for an assistant."""

    assistant_id: str
    """The ID of the assistant."""
    graph_id: str
    """The ID of the graph."""
    config: Config
    """The assistant config."""
    context: Context
    """The static context of the assistant."""
    created_at: datetime
    """The time the assistant was created."""
    metadata: Json
    """The assistant metadata."""
    version: int
    """The version of the assistant"""
    name: str
    """The name of the assistant"""
    description: str | None
    """The description of the assistant"""


class AssistantVersion(AssistantBase):
    """Represents a specific version of an assistant."""

    pass


class Assistant(AssistantBase):
    """Represents an assistant with additional properties."""

    updated_at: datetime
    """The last time the assistant was updated."""


class Interrupt(TypedDict):
    """Represents an interruption in the execution flow."""

    value: Any
    """The value associated with the interrupt."""
    id: str
    """The ID of the interrupt. Can be used to resume the interrupt."""


class Thread(TypedDict):
    """Represents a conversation thread."""

    thread_id: str
    """The ID of the thread."""
    created_at: datetime
    """The time the thread was created."""
    updated_at: datetime
    """The last time the thread was updated."""
    metadata: Json
    """The thread metadata."""
    status: ThreadStatus
    """The status of the thread, one of 'idle', 'busy', 'interrupted'."""
    values: Json
    """The current state of the thread."""
    interrupts: dict[str, list[Interrupt]]
    """Mapping of task ids to interrupts that were raised in that task."""


class ThreadTask(TypedDict):
    """Represents a task within a thread."""

    id: str
    name: str
    error: str | None
    interrupts: list[Interrupt]
    checkpoint: Checkpoint | None
    state: ThreadState | None
    result: dict[str, Any] | None


class ThreadState(TypedDict):
    """Represents the state of a thread."""

    values: list[dict] | dict[str, Any]
    """The state values."""
    next: Sequence[str]
    """The next nodes to execute. If empty, the thread is done until new input is 
    received."""
    checkpoint: Checkpoint
    """The ID of the checkpoint."""
    metadata: Json
    """Metadata for this state"""
    created_at: str | None
    """Timestamp of state creation"""
    parent_checkpoint: Checkpoint | None
    """The ID of the parent checkpoint. If missing, this is the root checkpoint."""
    tasks: Sequence[ThreadTask]
    """Tasks to execute in this step. If already attempted, may contain an error."""
    interrupts: list[Interrupt]
    """Interrupts which were thrown in this thread."""


class ThreadUpdateStateResponse(TypedDict):
    """Represents the response from updating a thread's state."""

    checkpoint: Checkpoint
    """Checkpoint of the latest state."""


class Run(TypedDict):
    """Represents a single execution run."""

    run_id: str
    """The ID of the run."""
    thread_id: str
    """The ID of the thread."""
    assistant_id: str
    """The assistant that was used for this run."""
    created_at: datetime
    """The time the run was created."""
    updated_at: datetime
    """The last time the run was updated."""
    status: RunStatus
    """The status of the run. One of 'pending', 'running', "error", 'success', "timeout", "interrupted"."""
    metadata: Json
    """The run metadata."""
    multitask_strategy: MultitaskStrategy
    """Strategy to handle concurrent runs on the same thread."""


class Cron(TypedDict):
    """Represents a scheduled task."""

    cron_id: str
    """The ID of the cron."""
    assistant_id: str
    """The ID of the assistant."""
    thread_id: str | None
    """The ID of the thread."""
    end_time: datetime | None
    """The end date to stop running the cron."""
    schedule: str
    """The schedule to run, cron format."""
    created_at: datetime
    """The time the cron was created."""
    updated_at: datetime
    """The last time the cron was updated."""
    payload: dict
    """The run payload to use for creating new run."""
    user_id: str | None
    """The user ID of the cron."""
    next_run_date: datetime | None
    """The next run date of the cron."""
    metadata: dict
    """The metadata of the cron."""


# Select field aliases for client-side typing of `select` parameters.
# These mirror the server's allowed field sets.

AssistantSelectField = Literal[
    "assistant_id",
    "graph_id",
    "name",
    "description",
    "config",
    "context",
    "created_at",
    "updated_at",
    "metadata",
    "version",
]

ThreadSelectField = Literal[
    "thread_id",
    "created_at",
    "updated_at",
    "metadata",
    "config",
    "context",
    "status",
    "values",
    "interrupts",
]

RunSelectField = Literal[
    "run_id",
    "thread_id",
    "assistant_id",
    "created_at",
    "updated_at",
    "status",
    "metadata",
    "kwargs",
    "multitask_strategy",
]

CronSelectField = Literal[
    "cron_id",
    "assistant_id",
    "thread_id",
    "end_time",
    "schedule",
    "created_at",
    "updated_at",
    "user_id",
    "payload",
    "next_run_date",
    "metadata",
    "now",
]

PrimitiveData = str | int | float | bool | None

QueryParamTypes = (
    Mapping[str, PrimitiveData | Sequence[PrimitiveData]]
    | list[tuple[str, PrimitiveData]]
    | tuple[tuple[str, PrimitiveData], ...]
    | str
    | bytes
)


class RunCreate(TypedDict):
    """Defines the parameters for initiating a background run."""

    thread_id: str | None
    """The identifier of the thread to run. If not provided, the run is stateless."""
    assistant_id: str
    """The identifier of the assistant to use for this run."""
    input: dict | None
    """Initial input data for the run."""
    metadata: dict | None
    """Additional metadata to associate with the run."""
    config: Config | None
    """Configuration options for the run."""
    context: Context | None
    """The static context of the run."""
    checkpoint_id: str | None
    """The identifier of a checkpoint to resume from."""
    interrupt_before: list[str] | None
    """List of node names to interrupt execution before."""
    interrupt_after: list[str] | None
    """List of node names to interrupt execution after."""
    webhook: str | None
    """URL to send webhook notifications about the run's progress."""
    multitask_strategy: MultitaskStrategy | None
    """Strategy for handling concurrent runs on the same thread."""


class Item(TypedDict):
    """Represents a single document or data entry in the graph's Store.

    Items are used to store cross-thread memories.
    """

    namespace: list[str]
    """The namespace of the item. A namespace is analogous to a document's directory."""
    key: str
    """The unique identifier of the item within its namespace.
    
    In general, keys needn't be globally unique.
    """
    value: dict[str, Any]
    """The value stored in the item. This is the document itself."""
    created_at: datetime
    """The timestamp when the item was created."""
    updated_at: datetime
    """The timestamp when the item was last updated."""


class ListNamespaceResponse(TypedDict):
    """Response structure for listing namespaces."""

    namespaces: list[list[str]]
    """A list of namespace paths, where each path is a list of strings."""


class SearchItem(Item, total=False):
    """Item with an optional relevance score from search operations.

    Attributes:
        score (Optional[float]): Relevance/similarity score. Included when
            searching a compatible store with a natural language query.
    """

    score: float | None


class SearchItemsResponse(TypedDict):
    """Response structure for searching items."""

    items: list[SearchItem]
    """A list of items matching the search criteria."""


class StreamPart(NamedTuple):
    """Represents a part of a stream response."""

    event: str
    """The type of event for this stream part."""
    data: dict
    """The data payload associated with the event."""


class Send(TypedDict):
    """Represents a message to be sent to a specific node in the graph.

    This type is used to explicitly send messages to nodes in the graph, typically
    used within Command objects to control graph execution flow.
    """

    node: str
    """The name of the target node to send the message to."""
    input: dict[str, Any] | None
    """Optional dictionary containing the input data to be passed to the node.

    If None, the node will be called with no input."""


class Command(TypedDict, total=False):
    """Represents one or more commands to control graph execution flow and state.

    This type defines the control commands that can be returned by nodes to influence
    graph execution. It lets you navigate to other nodes, update graph state,
    and resume from interruptions.
    """

    goto: Send | str | Sequence[Send | str]
    """Specifies where execution should continue. Can be:

        - A string node name to navigate to
        - A Send object to execute a node with specific input
        - A sequence of node names or Send objects to execute in order
    """
    update: dict[str, Any] | Sequence[tuple[str, Any]]
    """Updates to apply to the graph's state. Can be:

        - A dictionary of state updates to merge
        - A sequence of (key, value) tuples for ordered updates
    """
    resume: Any
    """Value to resume execution with after an interruption.
       Used in conjunction with interrupt() to implement control flow.
    """


class RunCreateMetadata(TypedDict):
    """Metadata for a run creation request."""

    run_id: str
    """The ID of the run."""

    thread_id: str | None
    """The ID of the thread."""

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/sdk-py/langgraph_sdk/sse.py
```py
"""Adapted from httpx_sse to split lines on \n, \r, \r\n per the SSE spec."""

from __future__ import annotations

from collections.abc import AsyncIterator, Iterator

import httpx
import orjson

from langgraph_sdk.schema import StreamPart

BytesLike = bytes | bytearray | memoryview


class BytesLineDecoder:
    """
    Handles incrementally reading lines from text.

    Has the same behaviour as the stdllib bytes splitlines,
    but handling the input iteratively.
    """

    def __init__(self) -> None:
        self.buffer = bytearray()
        self.trailing_cr: bool = False

    def decode(self, text: bytes) -> list[BytesLike]:
        # See https://docs.python.org/3/glossary.html#term-universal-newlines
        NEWLINE_CHARS = b"\n\r"

        # We always push a trailing `\r` into the next decode iteration.
        if self.trailing_cr:
            text = b"\r" + text
            self.trailing_cr = False
        if text.endswith(b"\r"):
            self.trailing_cr = True
            text = text[:-1]

        if not text:
            # NOTE: the edge case input of empty text doesn't occur in practice,
            # because other httpx internals filter out this value
            return []  # pragma: no cover

        trailing_newline = text[-1] in NEWLINE_CHARS
        lines = text.splitlines()

        if len(lines) == 1 and not trailing_newline:
            # No new lines, buffer the input and continue.
            self.buffer.extend(lines[0])
            return []

        if self.buffer:
            # Include any existing buffer in the first portion of the
            # splitlines result.
            self.buffer.extend(lines[0])
            lines = [self.buffer] + lines[1:]
            self.buffer = bytearray()

        if not trailing_newline:
            # If the last segment of splitlines is not newline terminated,
            # then drop it from our output and start a new buffer.
            self.buffer.extend(lines.pop())

        return lines

    def flush(self) -> list[BytesLike]:
        if not self.buffer and not self.trailing_cr:
            return []

        lines = [self.buffer]
        self.buffer = bytearray()
        self.trailing_cr = False
        return lines


class SSEDecoder:
    def __init__(self) -> None:
        self._event = ""
        self._data = bytearray()
        self._last_event_id = ""
        self._retry: int | None = None

    @property
    def last_event_id(self) -> str | None:
        """Return the last event identifier that was seen."""

        return self._last_event_id or None

    def decode(self, line: bytes) -> StreamPart | None:
        # See: https://html.spec.whatwg.org/multipage/server-sent-events.html#event-stream-interpretation  # noqa: E501

        if not line:
            if (
                not self._event
                and not self._data
                and not self._last_event_id
                and self._retry is None
            ):
                return None

            sse = StreamPart(
                event=self._event,
                data=orjson.loads(self._data) if self._data else None,  # type: ignore[invalid-argument-type]
            )

            # NOTE: as per the SSE spec, do not reset last_event_id.
            self._event = ""
            self._data = bytearray()
            self._retry = None

            return sse

        if line.startswith(b":"):
            return None

        fieldname, _, value = line.partition(b":")

        if value.startswith(b" "):
            value = value[1:]

        if fieldname == b"event":
            self._event = value.decode()
        elif fieldname == b"data":
            self._data.extend(value)
        elif fieldname == b"id":
            if b"\0" in value:
                pass
            else:
                self._last_event_id = value.decode()
        elif fieldname == b"retry":
            try:
                self._retry = int(value)
            except (TypeError, ValueError):
                pass
        else:
            pass  # Field is ignored.

        return None


async def aiter_lines_raw(response: httpx.Response) -> AsyncIterator[BytesLike]:
    decoder = BytesLineDecoder()
    async for chunk in response.aiter_bytes():
        for line in decoder.decode(chunk):
            yield line
    for line in decoder.flush():
        yield line


def iter_lines_raw(response: httpx.Response) -> Iterator[BytesLike]:
    decoder = BytesLineDecoder()
    for chunk in response.iter_bytes():
        for line in decoder.decode(chunk):
            yield line
    for line in decoder.flush():
        yield line

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/sdk-py/langgraph_sdk/auth/__init__.py
```py
from __future__ import annotations

import inspect
import typing
from collections.abc import Callable, Sequence

from langgraph_sdk.auth import exceptions, types

TH = typing.TypeVar("TH", bound=types.Handler)
AH = typing.TypeVar("AH", bound=types.Authenticator)


class Auth:
    """Add custom authentication and authorization management to your LangGraph application.

    The Auth class provides a unified system for handling authentication and
    authorization in LangGraph applications. It supports custom user authentication
    protocols and fine-grained authorization rules for different resources and
    actions.

    To use, create a separate python file and add the path to the file to your
    LangGraph API configuration file (`langgraph.json`). Within that file, create
    an instance of the Auth class and register authentication and authorization
    handlers as needed.

    Example `langgraph.json` file:

    ```json
    {
      "dependencies": ["."],
      "graphs": {
        "agent": "./my_agent/agent.py:graph"
      },
      "env": ".env",
      "auth": {
        "path": "./auth.py:my_auth"
      }
    ```

    Then the LangGraph server will load your auth file and run it server-side whenever a request comes in.

    ???+ example "Basic Usage"

        ```python
        from langgraph_sdk import Auth

        my_auth = Auth()

        async def verify_token(token: str) -> str:
            # Verify token and return user_id
            # This would typically be a call to your auth server
            return "user_id"

        @auth.authenticate
        async def authenticate(authorization: str) -> str:
            # Verify token and return user_id
            result = await verify_token(authorization)
            if result != "user_id":
                raise Auth.exceptions.HTTPException(
                    status_code=401, detail="Unauthorized"
                )
            return result

        # Global fallback handler
        @auth.on
        async def authorize_default(params: Auth.on.value):
            return False # Reject all requests (default behavior)

        @auth.on.threads.create
        async def authorize_thread_create(params: Auth.on.threads.create.value):
            # Allow the allowed user to create a thread
            assert params.get("metadata", {}).get("owner") == "allowed_user"

        @auth.on.store
        async def authorize_store(ctx: Auth.types.AuthContext, value: Auth.types.on):
            assert ctx.user.identity in value["namespace"], "Not authorized"
        ```

    ???+ note "Request Processing Flow"

        1. Authentication (your `@auth.authenticate` handler) is performed first on **every request**
        2. For authorization, the most specific matching handler is called:
            * If a handler exists for the exact resource and action, it is used (e.g., `@auth.on.threads.create`)
            * Otherwise, if a handler exists for the resource with any action, it is used (e.g., `@auth.on.threads`)
            * Finally, if no specific handlers match, the global handler is used (e.g., `@auth.on`)
            * If no global handler is set, the request is accepted

        This allows you to set default behavior with a global handler while
        overriding specific routes as needed.
    """

    __slots__ = (
        "on",
        "_handlers",
        "_global_handlers",
        "_authenticate_handler",
        "_handler_cache",
    )
    types = types
    """Reference to auth type definitions.
    
    Provides access to all type definitions used in the auth system,
    like ThreadsCreate, AssistantsRead, etc."""

    exceptions = exceptions
    """Reference to auth exception definitions.
    
    Provides access to all exception definitions used in the auth system,
    like HTTPException, etc.    
    """

    def __init__(self) -> None:
        self.on = _On(self)
        """Entry point for authorization handlers that control access to specific resources.

        The on class provides a flexible way to define authorization rules for different
        resources and actions in your application. It supports three main usage patterns:

        1. Global handlers that run for all resources and actions
        2. Resource-specific handlers that run for all actions on a resource
        3. Resource and action specific handlers for fine-grained control

        Each handler must be an async function that accepts two parameters:
            - ctx (AuthContext): Contains request context and authenticated user info
            - value: The data being authorized (type varies by endpoint)

        The handler should return one of:

            - None or True: Accept the request
            - False: Reject with 403 error
            - FilterType: Apply filtering rules to the response
        
        ???+ example "Examples"

            Global handler for all requests:

            ```python
            @auth.on
            async def reject_unhandled_requests(ctx: AuthContext, value: Any) -> None:
                print(f"Request to {ctx.path} by {ctx.user.identity}")
                return False
            ```

            Resource-specific handler. This would take precedence over the global handler
            for all actions on the `threads` resource:
            
            ```python
            @auth.on.threads
            async def check_thread_access(ctx: AuthContext, value: Any) -> bool:
                # Allow access only to threads created by the user
                return value.get("created_by") == ctx.user.identity
            ```

            Resource and action specific handler:

            ```python
            @auth.on.threads.delete
            async def prevent_thread_deletion(ctx: AuthContext, value: Any) -> bool:
                # Only admins can delete threads
                return "admin" in ctx.user.permissions
            ```

            Multiple resources or actions:

            ```python
            @auth.on(resources=["threads", "runs"], actions=["create", "update"])
            async def rate_limit_writes(ctx: AuthContext, value: Any) -> bool:
                # Implement rate limiting for write operations
                return await check_rate_limit(ctx.user.identity)
            ```

            Auth for the `store` resource is a bit different since its structure is developer defined.
            You typically want to enforce user creds in the namespace.

            ```python
            @auth.on.store
            async def check_store_access(ctx: AuthContext, value: Auth.types.on) -> bool:
                # Assuming you structure your store like (store.aput((user_id, application_context), key, value))
                assert value["namespace"][0] == ctx.user.identity
            ```
        """
        # These are accessed by the API. Changes to their names or types is
        # will be considered a breaking change.
        self._handlers: dict[tuple[str, str], list[types.Handler]] = {}
        self._global_handlers: list[types.Handler] = []
        self._authenticate_handler: types.Authenticator | None = None
        self._handler_cache: dict[tuple[str, str], types.Handler] = {}

    def authenticate(self, fn: AH) -> AH:
        """Register an authentication handler function.

        The authentication handler is responsible for verifying credentials
        and returning user scopes. It can accept any of the following parameters
        by name:

            - request (Request): The raw ASGI request object
            - path (str): The request path, e.g., "/threads/abcd-1234-abcd-1234/runs/abcd-1234-abcd-1234/stream"
            - method (str): The HTTP method, e.g., "GET"
            - path_params (dict[str, str]): URL path parameters, e.g., {"thread_id": "abcd-1234-abcd-1234", "run_id": "abcd-1234-abcd-1234"}
            - query_params (dict[str, str]): URL query parameters, e.g., {"stream": "true"}
            - headers (dict[bytes, bytes]): Request headers
            - authorization (str | None): The Authorization header value (e.g., "Bearer <token>")

        Args:
            fn: The authentication handler function to register.
                Must return a representation of the user. This could be a:
                    - string (the user id)
                    - dict containing {"identity": str, "permissions": list[str]}
                    - or an object with identity and permissions properties
                Permissions can be optionally used by your handlers downstream.

        Returns:
            The registered handler function.

        Raises:
            ValueError: If an authentication handler is already registered.

        ???+ example "Examples"

            Basic token authentication:

            ```python
            @auth.authenticate
            async def authenticate(authorization: str) -> str:
                user_id = verify_token(authorization)
                return user_id
            ```

            Accept the full request context:

            ```python
            @auth.authenticate
            async def authenticate(
                method: str,
                path: str,
                headers: dict[str, bytes]
            ) -> str:
                user = await verify_request(method, path, headers)
                return user
            ```

            Return user name and permissions:

            ```python
            @auth.authenticate
            async def authenticate(
                method: str,
                path: str,
                headers: dict[str, bytes]
            ) -> Auth.types.MinimalUserDict:
                permissions, user = await verify_request(method, path, headers)
                # Permissions could be things like ["runs:read", "runs:write", "threads:read", "threads:write"]
                return {
                    "identity": user["id"],
                    "permissions": permissions,
                    "display_name": user["name"],
                }
            ```
        """
        if self._authenticate_handler is not None:
            raise ValueError(
                f"Authentication handler already set as {self._authenticate_handler}."
            )
        self._authenticate_handler = fn
        return fn


## Helper types & utilities

V = typing.TypeVar("V", contravariant=True)


class _ActionHandler(typing.Protocol[V]):
    async def __call__(
        self, *, ctx: types.AuthContext, value: V
    ) -> types.HandlerResult: ...


T = typing.TypeVar("T", covariant=True)


class _ResourceActionOn(typing.Generic[T]):
    def __init__(
        self,
        auth: Auth,
        resource: typing.Literal["threads", "crons", "assistants"],
        action: typing.Literal[
            "create", "read", "update", "delete", "search", "create_run"
        ],
        value: type[T],
    ) -> None:
        self.auth = auth
        self.resource = resource
        self.action = action
        self.value = value

    def __call__(self, fn: _ActionHandler[T]) -> _ActionHandler[T]:
        _validate_handler(fn)
        _register_handler(self.auth, self.resource, self.action, fn)
        return fn


VCreate = typing.TypeVar("VCreate", covariant=True)
VUpdate = typing.TypeVar("VUpdate", covariant=True)
VRead = typing.TypeVar("VRead", covariant=True)
VDelete = typing.TypeVar("VDelete", covariant=True)
VSearch = typing.TypeVar("VSearch", covariant=True)


class _ResourceOn(typing.Generic[VCreate, VRead, VUpdate, VDelete, VSearch]):
    """
    Generic base class for resource-specific handlers.
    """

    value: type[VCreate | VUpdate | VRead | VDelete | VSearch]

    Create: type[VCreate]
    Read: type[VRead]
    Update: type[VUpdate]
    Delete: type[VDelete]
    Search: type[VSearch]

    def __init__(
        self,
        auth: Auth,
        resource: typing.Literal["threads", "crons", "assistants"],
    ) -> None:
        self.auth = auth
        self.resource = resource
        self.create: _ResourceActionOn[VCreate] = _ResourceActionOn(
            auth, resource, "create", self.Create
        )
        self.read: _ResourceActionOn[VRead] = _ResourceActionOn(
            auth, resource, "read", self.Read
        )
        self.update: _ResourceActionOn[VUpdate] = _ResourceActionOn(
            auth, resource, "update", self.Update
        )
        self.delete: _ResourceActionOn[VDelete] = _ResourceActionOn(
            auth, resource, "delete", self.Delete
        )
        self.search: _ResourceActionOn[VSearch] = _ResourceActionOn(
            auth, resource, "search", self.Search
        )

    @typing.overload
    def __call__(
        self,
        fn: (
            _ActionHandler[VCreate | VUpdate | VRead | VDelete | VSearch]
            | _ActionHandler[dict[str, typing.Any]]
        ),
    ) -> _ActionHandler[VCreate | VUpdate | VRead | VDelete | VSearch]: ...

    @typing.overload
    def __call__(
        self,
        *,
        resources: str | Sequence[str],
        actions: str | Sequence[str] | None = None,
    ) -> Callable[
        [_ActionHandler[VCreate | VUpdate | VRead | VDelete | VSearch]],
        _ActionHandler[VCreate | VUpdate | VRead | VDelete | VSearch],
    ]: ...

    def __call__(
        self,
        fn: (
            _ActionHandler[VCreate | VUpdate | VRead | VDelete | VSearch]
            | _ActionHandler[dict[str, typing.Any]]
            | None
        ) = None,
        *,
        resources: str | Sequence[str] | None = None,
        actions: str | Sequence[str] | None = None,
    ) -> (
        _ActionHandler[VCreate | VUpdate | VRead | VDelete | VSearch]
        | Callable[
            [_ActionHandler[VCreate | VUpdate | VRead | VDelete | VSearch]],
            _ActionHandler[VCreate | VUpdate | VRead | VDelete | VSearch],
        ]
    ):
        if fn is not None:
            _validate_handler(fn)
            return typing.cast(
                _ActionHandler[VCreate | VUpdate | VRead | VDelete | VSearch],
                _register_handler(self.auth, self.resource, "*", fn),
            )

        def decorator(
            handler: _ActionHandler[VCreate | VUpdate | VRead | VDelete | VSearch],
        ) -> _ActionHandler[VCreate | VUpdate | VRead | VDelete | VSearch]:
            _validate_handler(handler)
            return typing.cast(
                _ActionHandler[VCreate | VUpdate | VRead | VDelete | VSearch],
                _register_handler(self.auth, self.resource, "*", handler),
            )

        # Accept keyword-only parameters for future filtering behavior; referenced to satisfy linters.
        _ = resources, actions
        return decorator


class _AssistantsOn(
    _ResourceOn[
        types.AssistantsCreate,
        types.AssistantsRead,
        types.AssistantsUpdate,
        types.AssistantsDelete,
        types.AssistantsSearch,
    ]
):
    value = (
        types.AssistantsCreate
        | types.AssistantsRead
        | types.AssistantsUpdate
        | types.AssistantsDelete
        | types.AssistantsSearch
    )
    Create = types.AssistantsCreate
    Read = types.AssistantsRead
    Update = types.AssistantsUpdate
    Delete = types.AssistantsDelete
    Search = types.AssistantsSearch


class _ThreadsOn(
    _ResourceOn[
        types.ThreadsCreate,
        types.ThreadsRead,
        types.ThreadsUpdate,
        types.ThreadsDelete,
        types.ThreadsSearch,
    ]
):
    value = (
        types.ThreadsCreate
        | types.ThreadsRead
        | types.ThreadsUpdate
        | types.ThreadsDelete
        | types.ThreadsSearch
        | types.RunsCreate
    )
    Create = types.ThreadsCreate
    Read = types.ThreadsRead
    Update = types.ThreadsUpdate
    Delete = types.ThreadsDelete
    Search = types.ThreadsSearch
    CreateRun = types.RunsCreate

    def __init__(
        self,
        auth: Auth,
        resource: typing.Literal["threads", "crons", "assistants"],
    ) -> None:
        super().__init__(auth, resource)
        self.create_run: _ResourceActionOn[types.RunsCreate] = _ResourceActionOn(
            auth, resource, "create_run", self.CreateRun
        )


class _CronsOn(
    _ResourceOn[
        types.CronsCreate,
        types.CronsRead,
        types.CronsUpdate,
        types.CronsDelete,
        types.CronsSearch,
    ]
):
    value = type[
        types.CronsCreate
        | types.CronsRead
        | types.CronsUpdate
        | types.CronsDelete
        | types.CronsSearch
    ]

    Create = types.CronsCreate
    Read = types.CronsRead
    Update = types.CronsUpdate
    Delete = types.CronsDelete
    Search = types.CronsSearch


class _StoreOn:
    def __init__(self, auth: Auth) -> None:
        self._auth = auth

    @typing.overload
    def __call__(
        self,
        *,
        actions: (
            typing.Literal["put", "get", "search", "list_namespaces", "delete"]
            | Sequence[
                typing.Literal["put", "get", "search", "list_namespaces", "delete"]
            ]
            | None
        ) = None,
    ) -> Callable[[AHO], AHO]: ...

    @typing.overload
    def __call__(self, fn: AHO) -> AHO: ...

    def __call__(
        self,
        fn: AHO | None = None,
        *,
        actions: (
            typing.Literal["put", "get", "search", "list_namespaces", "delete"]
            | Sequence[
                typing.Literal["put", "get", "search", "list_namespaces", "delete"]
            ]
            | None
        ) = None,
    ) -> AHO | Callable[[AHO], AHO]:
        """Register a handler for specific resources and actions.

        Can be used as a decorator or with explicit resource/action parameters:

        @auth.on.store
        async def handler(): ... # Handle all store ops

        @auth.on.store(actions=("put", "get", "search", "delete"))
        async def handler(): ... # Handle specific store ops

        @auth.on.store.put
        async def handler(): ... # Handle store.put ops
        """
        if fn is not None:
            # Used as a plain decorator
            _register_handler(self._auth, "store", None, fn)
            return fn

        # Used with parameters, return a decorator
        def decorator(
            handler: AHO,
        ) -> AHO:
            if isinstance(actions, str):
                action_list = [actions]
            else:
                action_list = list(actions) if actions is not None else ["*"]
            for action in action_list:
                _register_handler(self._auth, "store", action, handler)
            return handler

        return decorator


AHO = typing.TypeVar("AHO", bound=_ActionHandler[dict[str, typing.Any]])


class _On:
    """Entry point for authorization handlers that control access to specific resources.

    The _On class provides a flexible way to define authorization rules for different resources
    and actions in your application. It supports three main usage patterns:

    1. Global handlers that run for all resources and actions
    2. Resource-specific handlers that run for all actions on a resource
    3. Resource and action specific handlers for fine-grained control

    Each handler must be an async function that accepts two parameters:
    - ctx (AuthContext): Contains request context and authenticated user info
    - value: The data being authorized (type varies by endpoint)

    The handler should return one of:
        - None or True: Accept the request
        - False: Reject with 403 error
        - FilterType: Apply filtering rules to the response

    ???+ example "Examples"

        Global handler for all requests:

        ```python
        @auth.on
        async def log_all_requests(ctx: AuthContext, value: Any) -> None:
            print(f"Request to {ctx.path} by {ctx.user.identity}")
            return True
        ```

        Resource-specific handler:

        ```python
        @auth.on.threads
        async def check_thread_access(ctx: AuthContext, value: Any) -> bool:
            # Allow access only to threads created by the user
            return value.get("created_by") == ctx.user.identity
        ```

        Resource and action specific handler:

        ```python
        @auth.on.threads.delete
        async def prevent_thread_deletion(ctx: AuthContext, value: Any) -> bool:
            # Only admins can delete threads
            return "admin" in ctx.user.permissions
        ```

        Multiple resources or actions:

        ```python
        @auth.on(resources=["threads", "runs"], actions=["create", "update"])
        async def rate_limit_writes(ctx: AuthContext, value: Any) -> bool:
            # Implement rate limiting for write operations
            return await check_rate_limit(ctx.user.identity)
        ```
    """

    __slots__ = (
        "_auth",
        "assistants",
        "threads",
        "runs",
        "crons",
        "store",
        "value",
    )

    def __init__(self, auth: Auth) -> None:
        self._auth = auth
        self.assistants = _AssistantsOn(auth, "assistants")
        self.threads = _ThreadsOn(auth, "threads")
        self.crons = _CronsOn(auth, "crons")
        self.store = _StoreOn(auth)
        self.value = dict[str, typing.Any]

    @typing.overload
    def __call__(
        self,
        *,
        resources: str | Sequence[str],
        actions: str | Sequence[str] | None = None,
    ) -> Callable[[AHO], AHO]: ...

    @typing.overload
    def __call__(self, fn: AHO) -> AHO: ...

    def __call__(
        self,
        fn: AHO | None = None,
        *,
        resources: str | Sequence[str] | None = None,
        actions: str | Sequence[str] | None = None,
    ) -> AHO | Callable[[AHO], AHO]:
        """Register a handler for specific resources and actions.

        Can be used as a decorator or with explicit resource/action parameters:

        @auth.on
        async def handler(): ...  # Global handler

        @auth.on(resources="threads")
        async def handler(): ...  # types.Handler for all thread actions

        @auth.on(resources="threads", actions="create")
        async def handler(): ...  # types.Handler for thread creation
        """
        if fn is not None:
            # Used as a plain decorator
            _register_handler(self._auth, None, None, fn)
            return fn

        # Used with parameters, return a decorator
        def decorator(
            handler: AHO,
        ) -> AHO:
            if isinstance(resources, str):
                resource_list = [resources]
            else:
                resource_list = list(resources) if resources is not None else ["*"]

            if isinstance(actions, str):
                action_list = [actions]
            else:
                action_list = list(actions) if actions is not None else ["*"]
            for resource in resource_list:
                for action in action_list:
                    _register_handler(self._auth, resource, action, handler)
            return handler

        return decorator


def _register_handler(
    auth: Auth,
    resource: str | None,
    action: str | None,
    fn: types.Handler,
) -> types.Handler:
    _validate_handler(fn)
    resource = resource or "*"
    action = action or "*"
    if resource == "*" and action == "*":
        if auth._global_handlers:
            raise ValueError("Global handler already set.")
        auth._global_handlers.append(fn)
    else:
        r = resource if resource is not None else "*"
        a = action if action is not None else "*"
        if (r, a) in auth._handlers:
            raise ValueError(f"types.Handler already set for {r}, {a}.")
        auth._handlers[(r, a)] = [fn]
    return fn


def _validate_handler(fn: Callable[..., typing.Any]) -> None:
    """Validates that an auth handler function meets the required signature.

    Auth handlers must:
    1. Be async functions
    2. Accept a ctx parameter of type AuthContext
    3. Accept a value parameter for the data being authorized
    """
    if not inspect.iscoroutinefunction(fn):
        raise ValueError(
            f"Auth handler '{getattr(fn, '__name__', fn)}' must be an async function. "
            "Add 'async' before 'def' to make it asynchronous and ensure"
            " any IO operations are non-blocking."
        )

    sig = inspect.signature(fn)
    if "ctx" not in sig.parameters:
        raise ValueError(
            f"Auth handler '{getattr(fn, '__name__', fn)}' must have a 'ctx: AuthContext' parameter. "
            "Update the function signature to include this required parameter."
        )
    if "value" not in sig.parameters:
        raise ValueError(
            f"Auth handler '{getattr(fn, '__name__', fn)}' must have a 'value' parameter. "
            " The value contains the mutable data being sent to the endpoint."
            "Update the function signature to include this required parameter."
        )


def is_studio_user(
    user: types.MinimalUser | types.BaseUser | types.MinimalUserDict,
) -> bool:
    return (
        isinstance(user, types.StudioUser)
        or isinstance(user, dict)
        and user.get("kind") == "StudioUser"  # ty: ignore[invalid-argument-type]
    )


__all__ = ["Auth", "types", "exceptions"]

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/sdk-py/langgraph_sdk/auth/exceptions.py
```py
"""Exceptions used in the auth system."""

from __future__ import annotations

import http
from collections.abc import Mapping


class HTTPException(Exception):
    """HTTP exception that you can raise to return a specific HTTP error response.

    Since this is defined in the auth module, we default to a 401 status code.

    Args:
        status_code: HTTP status code for the error. Defaults to 401 "Unauthorized".
        detail: Detailed error message. If `None`, uses a default
            message based on the status code.
        headers: Additional HTTP headers to include in the error response.

    Example:
        Default:
        ```python
        raise HTTPException()
        # HTTPException(status_code=401, detail='Unauthorized')
        ```

        Add headers:
        ```python
        raise HTTPException(headers={"X-Custom-Header": "Custom Value"})
        # HTTPException(status_code=401, detail='Unauthorized', headers={"WWW-Authenticate": "Bearer"})
        ```

        Custom error:
        ```python
        raise HTTPException(status_code=404, detail="Not found")
        ```
    """

    def __init__(
        self,
        status_code: int = 401,
        detail: str | None = None,
        headers: Mapping[str, str] | None = None,
    ) -> None:
        if detail is None:
            detail = http.HTTPStatus(status_code).phrase
        self.status_code = status_code
        self.detail = detail
        self.headers = headers

    def __str__(self) -> str:
        return f"{self.status_code}: {self.detail}"

    def __repr__(self) -> str:
        class_name = self.__class__.__name__
        return f"{class_name}(status_code={self.status_code!r}, detail={self.detail!r})"


__all__ = ["HTTPException"]

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/sdk-py/langgraph_sdk/auth/types.py
```py
"""Authentication and authorization types for LangGraph.

This module defines the core types used for authentication, authorization, and
request handling in LangGraph. It includes user protocols, authentication contexts,
and typed dictionaries for various API operations.

Note:
    All typing.TypedDict classes use total=False to make all fields typing.Optional by default.
"""

from __future__ import annotations

import typing
from collections.abc import Awaitable, Callable, Sequence
from dataclasses import dataclass
from datetime import datetime
from uuid import UUID

import typing_extensions

RunStatus = typing.Literal["pending", "error", "success", "timeout", "interrupted"]
"""Status of a run execution.

Values:
    - pending: Run is queued or in progress
    - error: Run failed with an error
    - success: Run completed successfully  
    - timeout: Run exceeded time limit
    - interrupted: Run was manually interrupted
"""

MultitaskStrategy = typing.Literal["reject", "rollback", "interrupt", "enqueue"]
"""Strategy for handling multiple concurrent tasks.

Values:
    - reject: Reject new tasks while one is in progress
    - rollback: Cancel current task and start new one
    - interrupt: Interrupt current task and start new one
    - enqueue: Queue new tasks to run after current one
"""

OnConflictBehavior = typing.Literal["raise", "do_nothing"]
"""Behavior when encountering conflicts.

Values:
    - raise: Raise an exception on conflict
    - do_nothing: Silently ignore conflicts
"""

IfNotExists = typing.Literal["create", "reject"]
"""Behavior when an entity doesn't exist.

Values:
    - create: Create the entity
    - reject: Reject the operation
"""

FilterType = (
    dict[
        str,
        str
        | dict[typing.Literal["$eq", "$contains"], str]
        | dict[typing.Literal["$contains"], list[str]],
    ]
    | dict[str, str]
)
"""Response type for authorization handlers.

Supports exact matches and operators:
    - Exact match shorthand: {"field": "value"}
    - Exact match: {"field": {"$eq": "value"}}
    - Contains (membership): {"field": {"$contains": "value"}}
    - Contains (subset containment): {"field": {"$contains": ["value1", "value2"]}}

Subset containment is only supported by newer versions of the LangGraph dev server;
install langgraph-runtime-inmem >= 0.14.1 to use this filter variant.

???+ example "Examples"

    Simple exact match filter for the resource owner:

    ```python
    filter = {"owner": "user-abcd123"}
    ```

    Explicit version of the exact match filter:

    ```python
    filter = {"owner": {"$eq": "user-abcd123"}}
    ```

    Containment (membership of a single element):

    ```python
    filter = {"participants": {"$contains": "user-abcd123"}}
    ```

    Containment (subset containment; all values must be present, but order doesn't matter):

    ```python
    filter = {"participants": {"$contains": ["user-abcd123", "user-efgh456"]}}
    ```

    Combining filters (treated as a logical `AND`):

    ```python
    filter = {"owner": "user-abcd123", "participants": {"$contains": "user-efgh456"}}
    ```
"""

ThreadStatus = typing.Literal["idle", "busy", "interrupted", "error"]
"""Status of a thread.

Values:
    - idle: Thread is available for work
    - busy: Thread is currently processing
    - interrupted: Thread was interrupted
    - error: Thread encountered an error
"""

MetadataInput = dict[str, typing.Any]
"""Type for arbitrary metadata attached to entities.

Allows storing custom key-value pairs with any entity.
Keys must be strings, values can be any JSON-serializable type.

???+ example "Examples"

    ```python
    metadata = {
        "created_by": "user123",
        "priority": 1,
        "tags": ["important", "urgent"]
    }
    ```
"""

HandlerResult = None | bool | FilterType
"""The result of a handler can be:
    * None | True: accept the request.
    * False: reject the request with a 403 error
    * FilterType: filter to apply
"""

Handler = Callable[..., Awaitable[HandlerResult]]

T = typing.TypeVar("T")


@typing.runtime_checkable
class MinimalUser(typing.Protocol):
    """User objects must at least expose the identity property."""

    @property
    def identity(self) -> str:
        """The unique identifier for the user.

        This could be a username, email, or any other unique identifier used
        to distinguish between different users in the system.
        """
        ...


class MinimalUserDict(typing.TypedDict, total=False):
    """The dictionary representation of a user."""

    identity: typing_extensions.Required[str]
    """The required unique identifier for the user."""
    display_name: str
    """The typing.Optional display name for the user."""
    is_authenticated: bool
    """Whether the user is authenticated. Defaults to True."""
    permissions: Sequence[str]
    """A list of permissions associated with the user.
    
    You can use these in your `@auth.on` authorization logic to determine
    access permissions to different resources.
    """


@typing.runtime_checkable
class BaseUser(typing.Protocol):
    """The base ASGI user protocol"""

    @property
    def is_authenticated(self) -> bool:
        """Whether the user is authenticated."""
        ...

    @property
    def display_name(self) -> str:
        """The display name of the user."""
        ...

    @property
    def identity(self) -> str:
        """The unique identifier for the user."""
        ...

    @property
    def permissions(self) -> Sequence[str]:
        """The permissions associated with the user."""
        ...

    def __getitem__(self, key):
        """Get a key from your minimal user dict."""
        ...

    def __contains__(self, key):
        """Check if a property exists."""
        ...

    def __iter__(self):
        """Iterate over the keys of the user."""
        ...


class StudioUser:
    """A user object that's populated from authenticated requests from the LangGraph studio.

    Note: Studio auth can be disabled in your `langgraph.json` config.

    ```json
    {
      "auth": {
        "disable_studio_auth": true
      }
    }
    ```

    You can use `isinstance` checks in your authorization handlers (`@auth.on`) to control access specifically
    for developers accessing the instance from the LangGraph Studio UI.

    ???+ example "Examples"

        ```python
        @auth.on
        async def allow_developers(ctx: Auth.types.AuthContext, value: Any) -> None:
            if isinstance(ctx.user, Auth.types.StudioUser):
                return None
            ...
            return False
        ```
    """

    __slots__ = ("username", "_is_authenticated", "_permissions")

    def __init__(self, username: str, is_authenticated: bool = False) -> None:
        self.username = username
        self._is_authenticated = is_authenticated
        self._permissions = ["authenticated"] if is_authenticated else []

    @property
    def is_authenticated(self) -> bool:
        return self._is_authenticated

    @property
    def display_name(self) -> str:
        return self.username

    @property
    def identity(self) -> str:
        return self.username

    @property
    def permissions(self) -> Sequence[str]:
        return self._permissions


Authenticator = Callable[
    ...,
    Awaitable[
        MinimalUser
        | str
        | BaseUser
        | MinimalUserDict
        | typing.Mapping[str, typing.Any],
    ],
]
"""Type for authentication functions.

An authenticator can return either:
1. A string (user_id)
2. A dict containing {"identity": str, "permissions": list[str]}
3. An object with identity and permissions properties

Permissions can be used downstream by your authorization logic to determine
access permissions to different resources.

The authenticate decorator will automatically inject any of the following parameters
by name if they are included in your function signature:

Parameters:
    request (Request): The raw ASGI request object
    body (dict): The parsed request body
    path (str): The request path
    method (str): The HTTP method (GET, POST, etc.)
    path_params (dict[str, str] | None): URL path parameters
    query_params (dict[str, str] | None): URL query parameters
    headers (dict[str, bytes] | None): Request headers
    authorization (str | None): The Authorization header value (e.g. "Bearer <token>")

???+ example "Examples"

    Basic authentication with token:

    ```python
    from langgraph_sdk import Auth

    auth = Auth()

    @auth.authenticate
    async def authenticate1(authorization: str) -> Auth.types.MinimalUserDict:
        return await get_user(authorization)
    ```

    Authentication with multiple parameters:

    ```    
    @auth.authenticate
    async def authenticate2(
        method: str,
        path: str,
        headers: dict[str, bytes]
    ) -> Auth.types.MinimalUserDict:
        # Custom auth logic using method, path and headers
        user = verify_request(method, path, headers)
        return user
    ```

    Accepting the raw ASGI request:

    ```python
    MY_SECRET = "my-secret-key"
    @auth.authenticate
    async def get_current_user(request: Request) -> Auth.types.MinimalUserDict:
        try:
            token = (request.headers.get("authorization") or "").split(" ", 1)[1]
            payload = jwt.decode(token, MY_SECRET, algorithms=["HS256"])
        except (IndexError, InvalidTokenError):
            raise HTTPException(
                status_code=401,
                detail="Invalid token",
                headers={"WWW-Authenticate": "Bearer"},
            )

        async with httpx.AsyncClient() as client:
            response = await client.get(
                f"https://api.myauth-provider.com/auth/v1/user",
                headers={"Authorization": f"Bearer {MY_SECRET}"}
            )
            if response.status_code != 200:
                raise HTTPException(status_code=401, detail="User not found")
                
            user_data = response.json()
            return {
                "identity": user_data["id"],
                "display_name": user_data.get("name"),
                "permissions": user_data.get("permissions", []),
                "is_authenticated": True,
            }
    ```
"""


@dataclass(slots=True)
class BaseAuthContext:
    """Base class for authentication context.

    Provides the fundamental authentication information needed for
    authorization decisions.
    """

    permissions: Sequence[str]
    """The permissions granted to the authenticated user."""

    user: BaseUser
    """The authenticated user."""


@typing.final
@dataclass(slots=True)
class AuthContext(BaseAuthContext):
    """Complete authentication context with resource and action information.

    Extends BaseAuthContext with specific resource and action being accessed,
    allowing for fine-grained access control decisions.
    """

    resource: typing.Literal["runs", "threads", "crons", "assistants", "store"]
    """The resource being accessed."""

    action: typing.Literal[
        "create",
        "read",
        "update",
        "delete",
        "search",
        "create_run",
        "put",
        "get",
        "list_namespaces",
    ]
    """The action being performed on the resource.
    
    Most resources support the following actions:
    - create: Create a new resource
    - read: Read information about a resource
    - update: Update an existing resource
    - delete: Delete a resource
    - search: Search for resources

    The store supports the following actions:
    - put: Add or update a document in the store
    - get: Get a document from the store
    - list_namespaces: List the namespaces in the store
    """


class ThreadTTL(typing.TypedDict, total=False):
    """Time-to-live configuration for a thread.

    Matches the OpenAPI schema where TTL is represented as an object with
    an optional strategy and a time value in minutes.
    """

    strategy: typing.Literal["delete"]
    """TTL strategy. Currently only 'delete' is supported."""

    ttl: int
    """Time-to-live in minutes from now until the thread should be swept."""


class ThreadsCreate(typing.TypedDict, total=False):
    """Parameters for creating a new thread.

    ???+ example "Examples"

        ```python
        create_params = {
            "thread_id": UUID("123e4567-e89b-12d3-a456-426614174000"),
            "metadata": {"owner": "user123"},
            "if_exists": "do_nothing"
        }
        ```
    """

    thread_id: UUID
    """Unique identifier for the thread."""

    metadata: MetadataInput
    """typing.Optional metadata to attach to the thread."""

    if_exists: OnConflictBehavior
    """Behavior when a thread with the same ID already exists."""

    ttl: ThreadTTL
    """Optional TTL configuration for the thread."""


class ThreadsRead(typing.TypedDict, total=False):
    """Parameters for reading thread state or run information.

    This type is used in three contexts:
    1. Reading thread, thread version, or thread state information: Only thread_id is provided
    2. Reading run information: Both thread_id and run_id are provided
    """

    thread_id: UUID
    """Unique identifier for the thread."""

    run_id: UUID | None
    """Run ID to filter by. Only used when reading run information within a thread."""


class ThreadsUpdate(typing.TypedDict, total=False):
    """Parameters for updating a thread or run.

    Called for updates to a thread, thread version, or run
    cancellation.
    """

    thread_id: UUID
    """Unique identifier for the thread."""

    metadata: MetadataInput
    """typing.Optional metadata to update."""

    action: typing.Literal["interrupt", "rollback"] | None
    """typing.Optional action to perform on the thread."""


class ThreadsDelete(typing.TypedDict, total=False):
    """Parameters for deleting a thread.

    Called for deletes to a thread, thread version, or run
    """

    thread_id: UUID
    """Unique identifier for the thread."""

    run_id: UUID | None
    """typing.Optional run ID to filter by."""


class ThreadsSearch(typing.TypedDict, total=False):
    """Parameters for searching threads.

    Called for searches to threads or runs.
    """

    metadata: MetadataInput
    """typing.Optional metadata to filter by."""

    values: MetadataInput
    """typing.Optional values to filter by."""

    status: ThreadStatus | None
    """typing.Optional status to filter by."""

    limit: int
    """Maximum number of results to return."""

    offset: int
    """Offset for pagination."""

    ids: Sequence[UUID] | None
    """typing.Optional list of thread IDs to filter by."""

    thread_id: UUID | None
    """typing.Optional thread ID to filter by."""


class RunsCreate(typing.TypedDict, total=False):
    """Payload for creating a run.

    ???+ example "Examples"

        ```python
        create_params = {
            "assistant_id": UUID("123e4567-e89b-12d3-a456-426614174000"),
            "thread_id": UUID("123e4567-e89b-12d3-a456-426614174001"),
            "run_id": UUID("123e4567-e89b-12d3-a456-426614174002"),
            "status": "pending",
            "metadata": {"owner": "user123"},
            "prevent_insert_if_inflight": True,
            "multitask_strategy": "reject",
            "if_not_exists": "create",
            "after_seconds": 10,
            "kwargs": {"key": "value"},
            "action": "interrupt"
        }
        ```
    """

    assistant_id: UUID | None
    """typing.Optional assistant ID to use for this run."""

    thread_id: UUID | None
    """typing.Optional thread ID to use for this run."""

    run_id: UUID | None
    """typing.Optional run ID to use for this run."""

    status: RunStatus | None
    """typing.Optional status for this run."""

    metadata: MetadataInput
    """typing.Optional metadata for the run."""

    prevent_insert_if_inflight: bool
    """Prevent inserting a new run if one is already in flight."""

    multitask_strategy: MultitaskStrategy
    """Multitask strategy for this run."""

    if_not_exists: IfNotExists
    """IfNotExists for this run."""

    after_seconds: int
    """Number of seconds to wait before creating the run."""

    kwargs: dict[str, typing.Any]
    """Keyword arguments to pass to the run."""

    action: typing.Literal["interrupt", "rollback"] | None
    """Action to take if updating an existing run."""


class AssistantsCreate(typing.TypedDict, total=False):
    """Payload for creating an assistant.

    ???+ example "Examples"

        ```python
        create_params = {
            "assistant_id": UUID("123e4567-e89b-12d3-a456-426614174000"),
            "graph_id": "graph123",
            "config": {"tags": ["tag1", "tag2"]},
            "context": {"key": "value"},
            "metadata": {"owner": "user123"},
            "if_exists": "do_nothing",
            "name": "Assistant 1"
        }
        ```
    """

    assistant_id: UUID
    """Unique identifier for the assistant."""

    graph_id: str
    """Graph ID to use for this assistant."""

    config: dict[str, typing.Any]
    """typing.Optional configuration for the assistant."""

    context: dict[str, typing.Any]

    metadata: MetadataInput
    """typing.Optional metadata to attach to the assistant."""

    if_exists: OnConflictBehavior
    """Behavior when an assistant with the same ID already exists."""

    name: str
    """Name of the assistant."""


class AssistantsRead(typing.TypedDict, total=False):
    """Payload for reading an assistant.

    ???+ example "Examples"

        ```python
        read_params = {
            "assistant_id": UUID("123e4567-e89b-12d3-a456-426614174000"),
            "metadata": {"owner": "user123"}
        }
        ```
    """

    assistant_id: UUID
    """Unique identifier for the assistant."""

    metadata: MetadataInput
    """typing.Optional metadata to filter by."""


class AssistantsUpdate(typing.TypedDict, total=False):
    """Payload for updating an assistant.

    ???+ example "Examples"

        ```python
        update_params = {
            "assistant_id": UUID("123e4567-e89b-12d3-a456-426614174000"),
            "graph_id": "graph123",
            "config": {"tags": ["tag1", "tag2"]},
            "context": {"key": "value"},
            "metadata": {"owner": "user123"},
            "name": "Assistant 1",
            "version": 1
        }
        ```
    """

    assistant_id: UUID
    """Unique identifier for the assistant."""

    graph_id: str | None
    """typing.Optional graph ID to update."""

    config: dict[str, typing.Any]
    """typing.Optional configuration to update."""

    context: dict[str, typing.Any]
    """The static context of the assistant."""

    metadata: MetadataInput
    """typing.Optional metadata to update."""

    name: str | None
    """typing.Optional name to update."""

    version: int | None
    """typing.Optional version to update."""


class AssistantsDelete(typing.TypedDict):
    """Payload for deleting an assistant.

    ???+ example "Examples"

        ```python
        delete_params = {
            "assistant_id": UUID("123e4567-e89b-12d3-a456-426614174000")
        }
        ```
    """

    assistant_id: UUID
    """Unique identifier for the assistant."""


class AssistantsSearch(typing.TypedDict):
    """Payload for searching assistants.

    ???+ example "Examples"

        ```python
        search_params = {
            "graph_id": "graph123",
            "metadata": {"owner": "user123"},
            "limit": 10,
            "offset": 0
        }
        ```
    """

    graph_id: str | None
    """typing.Optional graph ID to filter by."""

    metadata: MetadataInput
    """typing.Optional metadata to filter by."""

    limit: int
    """Maximum number of results to return."""

    offset: int
    """Offset for pagination."""


class CronsCreate(typing.TypedDict, total=False):
    """Payload for creating a cron job.

    ???+ example "Examples"

        ```python
        create_params = {
            "payload": {"key": "value"},
            "schedule": "0 0 * * *",
            "cron_id": UUID("123e4567-e89b-12d3-a456-426614174000"),
            "thread_id": UUID("123e4567-e89b-12d3-a456-426614174001"),
            "user_id": "user123",
            "end_time": datetime(2024, 3, 16, 10, 0, 0)
        }
        ```
    """

    payload: dict[str, typing.Any]
    """Payload for the cron job."""

    schedule: str
    """Schedule for the cron job."""

    cron_id: UUID | None
    """typing.Optional unique identifier for the cron job."""

    thread_id: UUID | None
    """typing.Optional thread ID to use for this cron job."""

    user_id: str | None
    """typing.Optional user ID to use for this cron job."""

    end_time: datetime | None
    """typing.Optional end time for the cron job."""


class CronsDelete(typing.TypedDict):
    """Payload for deleting a cron job.

    ???+ example "Examples"

        ```python
        delete_params = {
            "cron_id": UUID("123e4567-e89b-12d3-a456-426614174000")
        }
        ```
    """

    cron_id: UUID
    """Unique identifier for the cron job."""


class CronsRead(typing.TypedDict):
    """Payload for reading a cron job.

    ???+ example "Examples"

        ```python
        read_params = {
            "cron_id": UUID("123e4567-e89b-12d3-a456-426614174000")
        }
        ```
    """

    cron_id: UUID
    """Unique identifier for the cron job."""


class CronsUpdate(typing.TypedDict, total=False):
    """Payload for updating a cron job.

    ???+ example "Examples"

        ```python
        update_params = {
            "cron_id": UUID("123e4567-e89b-12d3-a456-426614174000"),
            "payload": {"key": "value"},
            "schedule": "0 0 * * *"
        }
        ```
    """

    cron_id: UUID
    """Unique identifier for the cron job."""

    payload: dict[str, typing.Any] | None
    """typing.Optional payload to update."""

    schedule: str | None
    """typing.Optional schedule to update."""


class CronsSearch(typing.TypedDict, total=False):
    """Payload for searching cron jobs.

    ???+ example "Examples"

        ```python
        search_params = {
            "assistant_id": UUID("123e4567-e89b-12d3-a456-426614174000"),
            "thread_id": UUID("123e4567-e89b-12d3-a456-426614174001"),
            "limit": 10,
            "offset": 0
        }
        ```
    """

    assistant_id: UUID | None
    """typing.Optional assistant ID to filter by."""

    thread_id: UUID | None
    """typing.Optional thread ID to filter by."""

    limit: int
    """Maximum number of results to return."""

    offset: int
    """Offset for pagination."""


class StoreGet(typing.TypedDict):
    """Operation to retrieve a specific item by its namespace and key."""

    namespace: tuple[str, ...]
    """Hierarchical path that uniquely identifies the item's location."""

    key: str
    """Unique identifier for the item within its specific namespace."""


class StoreSearch(typing.TypedDict):
    """Operation to search for items within a specified namespace hierarchy."""

    namespace: tuple[str, ...]
    """Prefix filter for defining the search scope."""

    filter: dict[str, typing.Any] | None
    """Key-value pairs for filtering results based on exact matches or comparison operators."""

    limit: int
    """Maximum number of items to return in the search results."""

    offset: int
    """Number of matching items to skip for pagination."""

    query: str | None
    """Naturalj language search query for semantic search capabilities."""


class StoreListNamespaces(typing.TypedDict):
    """Operation to list and filter namespaces in the store."""

    namespace: tuple[str, ...] | None
    """Prefix filter namespaces."""

    suffix: tuple[str, ...] | None
    """Optional conditions for filtering namespaces."""

    max_depth: int | None
    """Maximum depth of namespace hierarchy to return.

    Note:
        Namespaces deeper than this level will be truncated.
    """

    limit: int
    """Maximum number of namespaces to return."""

    offset: int
    """Number of namespaces to skip for pagination."""


class StorePut(typing.TypedDict):
    """Operation to store, update, or delete an item in the store."""

    namespace: tuple[str, ...]
    """Hierarchical path that identifies the location of the item."""

    key: str
    """Unique identifier for the item within its namespace."""

    value: dict[str, typing.Any] | None
    """The data to store, or `None` to mark the item for deletion."""

    index: typing.Literal[False] | list[str] | None
    """Optional index configuration for full-text search."""


class StoreDelete(typing.TypedDict):
    """Operation to delete an item from the store."""

    namespace: tuple[str, ...]
    """Hierarchical path that uniquely identifies the item's location."""

    key: str
    """Unique identifier for the item within its specific namespace."""


class on:
    """Namespace for type definitions of different API operations.

    This class organizes type definitions for create, read, update, delete,
    and search operations across different resources (threads, assistants, crons).

    ???+ note "Usage"
        ```python
        from langgraph_sdk import Auth

        auth = Auth()

        @auth.on
        def handle_all(params: Auth.on.value):
            raise Exception("Not authorized")

        @auth.on.threads.create
        def handle_thread_create(params: Auth.on.threads.create.value):
            # Handle thread creation
            pass

        @auth.on.assistants.search
        def handle_assistant_search(params: Auth.on.assistants.search.value):
            # Handle assistant search
            pass
        ```
    """

    value = dict[str, typing.Any]

    class threads:
        """Types for thread-related operations."""

        value = (
            ThreadsCreate | ThreadsRead | ThreadsUpdate | ThreadsDelete | ThreadsSearch
        )

        class create:
            """Type for thread creation parameters."""

            value = ThreadsCreate

        class create_run:
            """Type for creating or streaming a run."""

            value = RunsCreate

        class read:
            """Type for thread read parameters."""

            value = ThreadsRead

        class update:
            """Type for thread update parameters."""

            value = ThreadsUpdate

        class delete:
            """Type for thread deletion parameters."""

            value = ThreadsDelete

        class search:
            """Type for thread search parameters."""

            value = ThreadsSearch

    class assistants:
        """Types for assistant-related operations."""

        value = (
            AssistantsCreate
            | AssistantsRead
            | AssistantsUpdate
            | AssistantsDelete
            | AssistantsSearch
        )

        class create:
            """Type for assistant creation parameters."""

            value = AssistantsCreate

        class read:
            """Type for assistant read parameters."""

            value = AssistantsRead

        class update:
            """Type for assistant update parameters."""

            value = AssistantsUpdate

        class delete:
            """Type for assistant deletion parameters."""

            value = AssistantsDelete

        class search:
            """Type for assistant search parameters."""

            value = AssistantsSearch

    class crons:
        """Types for cron-related operations."""

        value = CronsCreate | CronsRead | CronsUpdate | CronsDelete | CronsSearch

        class create:
            """Type for cron creation parameters."""

            value = CronsCreate

        class read:
            """Type for cron read parameters."""

            value = CronsRead

        class update:
            """Type for cron update parameters."""

            value = CronsUpdate

        class delete:
            """Type for cron deletion parameters."""

            value = CronsDelete

        class search:
            """Type for cron search parameters."""

            value = CronsSearch

    class store:
        """Types for store-related operations."""

        value = StoreGet | StoreSearch | StoreListNamespaces | StorePut | StoreDelete

        class put:
            """Type for store put parameters."""

            value = StorePut

        class get:
            """Type for store get parameters."""

            value = StoreGet

        class search:
            """Type for store search parameters."""

            value = StoreSearch

        class delete:
            """Type for store delete parameters."""

            value = StoreDelete

        class list_namespaces:
            """Type for store list namespaces parameters."""

            value = StoreListNamespaces


__all__ = [
    "on",
    "MetadataInput",
    "RunsCreate",
    "ThreadsCreate",
    "ThreadsRead",
    "ThreadsUpdate",
    "ThreadsDelete",
    "ThreadsSearch",
    "AssistantsCreate",
    "AssistantsRead",
    "AssistantsUpdate",
    "AssistantsDelete",
    "AssistantsSearch",
    "StoreGet",
    "StoreSearch",
    "StoreListNamespaces",
    "StorePut",
    "StoreDelete",
]

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/sdk-py/tests/test_api_parity.py
```py
from __future__ import annotations

import inspect
import re

import pytest

from langgraph_sdk.client import (
    AssistantsClient,
    CronClient,
    RunsClient,
    StoreClient,
    SyncAssistantsClient,
    SyncCronClient,
    SyncRunsClient,
    SyncStoreClient,
    SyncThreadsClient,
    ThreadsClient,
)


def _public_methods(cls) -> dict[str, object]:
    methods: dict[str, object] = {}
    # Use the raw class dict to avoid runtime wrappers from plugins/decorators
    for name, member in cls.__dict__.items():
        if name.startswith("_"):
            continue
        if inspect.isfunction(member):
            methods[name] = member
    return methods


def _strip_self(sig: inspect.Signature) -> inspect.Signature:
    params = list(sig.parameters.values())
    if params and params[0].name == "self":
        params = params[1:]
    return sig.replace(parameters=params)


def _normalize_return_annotation(ann: object) -> str:
    s = str(ann)
    s = re.sub(r"\s+", "", s)
    s = s.replace("typing.", "").replace("collections.abc.", "")
    s = re.sub(r"AsyncGenerator\[([^,\]]+)(?:,[^\]]*)?\]", r"Iterator[\1]", s)
    s = re.sub(r"Generator\[([^,\]]+)(?:,[^\]]*)?\]", r"Iterator[\1]", s)
    s = re.sub(r"AsyncIterator\[(.+)\]", r"Iterator[\1]", s)
    s = re.sub(r"AsyncIterable\[(.+)\]", r"Iterable[\1]", s)
    return s


@pytest.mark.parametrize(
    "async_cls,sync_cls",
    [
        (AssistantsClient, SyncAssistantsClient),
        (ThreadsClient, SyncThreadsClient),
        (RunsClient, SyncRunsClient),
        (CronClient, SyncCronClient),
        (StoreClient, SyncStoreClient),
    ],
)
def test_sync_api_matches_async(async_cls, sync_cls):
    async_methods = _public_methods(async_cls)
    sync_methods = _public_methods(sync_cls)

    # Method name parity
    assert set(sync_methods.keys()) == set(async_methods.keys()), (
        f"Method sets differ: async-only={set(async_methods) - set(sync_methods)}, sync-only={set(sync_methods) - set(async_methods)}"
    )

    for name, async_fn in async_methods.items():
        sync_fn = sync_methods[name]

        # Use inspect.signature for parameter names (robust across versions)
        async_sig = _strip_self(inspect.signature(async_fn))
        sync_sig = _strip_self(inspect.signature(sync_fn))

        a_names = list(async_sig.parameters.keys())
        s_names = list(sync_sig.parameters.keys())

        assert set(a_names) == set(s_names), (
            f"Parameter names differ for {async_cls.__name__}.{name}: "
            f"async={a_names}, sync={s_names}"
        )

        # Compare default presence and parameter kinds (with some tolerance)
        a_params = async_sig.parameters
        s_params = sync_sig.parameters

        def kinds_compatible(
            akind: inspect._ParameterKind, skind: inspect._ParameterKind
        ) -> bool:
            if akind == skind:
                return True
            return {
                inspect.Parameter.KEYWORD_ONLY,
                inspect.Parameter.POSITIONAL_OR_KEYWORD,
            } == {akind, skind}

        for pname in set(a_names) & set(s_names):
            apar = a_params[pname]
            spar = s_params[pname]
            assert kinds_compatible(apar.kind, spar.kind), (
                f"Parameter kind mismatch for {async_cls.__name__}.{name}.{pname}: "
                f"async={apar.kind}, sync={spar.kind}"
            )
            assert (apar.default is inspect._empty) == (
                spar.default is inspect._empty
            ), (
                f"Default presence mismatch for {async_cls.__name__}.{name}.{pname}: "
                f"async_has_default={apar.default is not inspect._empty}, "
                f"sync_has_default={spar.default is not inspect._empty}"
            )

        # Return annotations must match or be iterator-equivalent
        a_ret = _normalize_return_annotation(async_sig.return_annotation)
        s_ret = _normalize_return_annotation(sync_sig.return_annotation)
        assert a_ret == s_ret, (
            f"Return annotation mismatch for {async_cls.__name__}.{name}: "
            f"async={a_ret}, sync={s_ret}"
        )

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/sdk-py/tests/test_client_stream.py
```py
from __future__ import annotations

from collections.abc import Iterator, Sequence
from pathlib import Path

import httpx
import pytest

from langgraph_sdk.client import HttpClient, SyncHttpClient
from langgraph_sdk.schema import StreamPart
from langgraph_sdk.sse import BytesLike, BytesLineDecoder, SSEDecoder

with open(Path(__file__).parent / "fixtures" / "response.txt", "rb") as f:
    RESPONSE_PAYLOAD = f.read()


class AsyncListByteStream(httpx.AsyncByteStream):
    def __init__(self, chunks: Sequence[bytes], exc: Exception | None = None) -> None:
        self._chunks = list(chunks)
        self._exc = exc

    async def __aiter__(self):  # type: ignore[override]
        for chunk in self._chunks:
            yield chunk
        if self._exc is not None:
            raise self._exc

    async def aclose(self) -> None:
        return None


class ListByteStream(httpx.ByteStream):
    def __init__(self, chunks: Sequence[bytes], exc: Exception | None = None) -> None:
        self._chunks = list(chunks)
        self._exc = exc

    def __iter__(self):  # type: ignore[override]
        yield from self._chunks
        if self._exc is not None:
            raise self._exc

    def close(self) -> None:
        return None


def iter_lines_raw(payload: list[bytes]) -> Iterator[BytesLike]:
    decoder = BytesLineDecoder()
    for part in payload:
        yield from decoder.decode(part)
    yield from decoder.flush()


def test_stream_see():
    for groups in (
        [RESPONSE_PAYLOAD],
        RESPONSE_PAYLOAD.splitlines(keepends=True),
    ):
        parts: list[StreamPart] = []

        decoder = SSEDecoder()
        for line in iter_lines_raw(groups):
            sse = decoder.decode(line=line.rstrip(b"\n"))
            if sse is not None:
                parts.append(sse)
        if sse := decoder.decode(b""):
            parts.append(sse)

        assert decoder.decode(b"") is None
        assert len(parts) == 79


@pytest.mark.asyncio
async def test_http_client_stream_flushes_trailing_event():
    payload = b'event: foo\ndata: {"bar": 1}\n'

    async def handler(request: httpx.Request) -> httpx.Response:
        assert request.headers["accept"] == "text/event-stream"
        assert request.headers["cache-control"] == "no-store"
        return httpx.Response(
            200,
            headers={"Content-Type": "text/event-stream"},
            content=payload,
        )

    transport = httpx.MockTransport(handler)
    async with httpx.AsyncClient(
        transport=transport, base_url="https://example.com"
    ) as client:
        http_client = HttpClient(client)
        parts = [part async for part in http_client.stream("/stream", "GET")]

    assert parts == [StreamPart(event="foo", data={"bar": 1})]


def test_sync_http_client_stream_recovers_after_disconnect():
    reconnect_path = "/reconnect"
    first_chunks = [
        b"id: 1\n",
        b"event: values\n",
        b'data: {"step": 1}\n\n',
    ]
    second_chunks = [
        b"id: 2\n",
        b"event: values\n",
        b'data: {"step": 2}\n\n',
        b"event: end\n",
        b"data: null\n\n",
    ]
    call_count = 0

    def handler(request: httpx.Request) -> httpx.Response:
        nonlocal call_count
        call_count += 1
        if call_count == 1:
            assert request.method == "POST"
            assert request.url.path == "/stream"
            assert request.headers["accept"] == "text/event-stream"
            assert request.headers["cache-control"] == "no-store"
            assert "last-event-id" not in {
                k.lower(): v for k, v in request.headers.items()
            }
            assert request.read()
            return httpx.Response(
                200,
                headers={
                    "Content-Type": "text/event-stream",
                    "Location": reconnect_path,
                },
                stream=ListByteStream(
                    first_chunks,
                    httpx.RemoteProtocolError("incomplete chunked read"),
                ),
            )
        if call_count == 2:
            assert request.method == "GET"
            assert request.url.path == reconnect_path
            assert request.headers["Last-Event-ID"] == "1"
            assert request.read() == b""
            return httpx.Response(
                200,
                headers={"Content-Type": "text/event-stream"},
                stream=ListByteStream(second_chunks),
            )
        raise AssertionError("unexpected request")

    transport = httpx.MockTransport(handler)
    with httpx.Client(transport=transport, base_url="https://example.com") as client:
        http_client = SyncHttpClient(client)
        parts = list(http_client.stream("/stream", "POST", json={"payload": "value"}))

    assert call_count == 2
    assert parts == [
        StreamPart(event="values", data={"step": 1}),
        StreamPart(event="values", data={"step": 2}),
        StreamPart(event="end", data=None),
    ]


@pytest.mark.asyncio
async def test_http_client_stream_recovers_after_disconnect():
    reconnect_path = "/reconnect"
    first_chunks = [
        b"id: 1\n",
        b"event: values\n",
        b'data: {"step": 1}\n\n',
    ]
    second_chunks = [
        b"id: 2\n",
        b"event: values\n",
        b'data: {"step": 2}\n\n',
        b"event: end\n",
        b"data: null\n\n",
    ]
    call_count = 0

    async def handler(request: httpx.Request) -> httpx.Response:
        nonlocal call_count
        call_count += 1
        if call_count == 1:
            assert request.method == "POST"
            assert request.url.path == "/stream"
            assert request.headers["accept"] == "text/event-stream"
            assert request.headers["cache-control"] == "no-store"
            assert "last-event-id" not in {
                k.lower(): v for k, v in request.headers.items()
            }
            assert await request.aread()
            return httpx.Response(
                200,
                headers={
                    "Content-Type": "text/event-stream",
                    "Location": reconnect_path,
                },
                stream=AsyncListByteStream(
                    first_chunks,
                    httpx.RemoteProtocolError("incomplete chunked read"),
                ),
            )
        if call_count == 2:
            assert request.method == "GET"
            assert request.url.path == reconnect_path
            assert request.headers["Last-Event-ID"] == "1"
            assert await request.aread() == b""
            return httpx.Response(
                200,
                headers={"Content-Type": "text/event-stream"},
                stream=AsyncListByteStream(second_chunks),
            )
        raise AssertionError("unexpected request")

    transport = httpx.MockTransport(handler)
    async with httpx.AsyncClient(
        transport=transport, base_url="https://example.com"
    ) as client:
        http_client = HttpClient(client)
        parts = [
            part
            async for part in http_client.stream(
                "/stream", "POST", json={"payload": "value"}
            )
        ]

    assert call_count == 2
    assert parts == [
        StreamPart(event="values", data={"step": 1}),
        StreamPart(event="values", data={"step": 2}),
        StreamPart(event="end", data=None),
    ]


def test_sync_http_client_stream_flushes_trailing_event():
    payload = b'event: foo\ndata: {"bar": 1}\n'

    def handler(request: httpx.Request) -> httpx.Response:
        assert request.headers["accept"] == "text/event-stream"
        assert request.headers["cache-control"] == "no-store"
        return httpx.Response(
            200,
            headers={"Content-Type": "text/event-stream"},
            content=payload,
        )

    transport = httpx.MockTransport(handler)
    with httpx.Client(transport=transport, base_url="https://example.com") as client:
        http_client = SyncHttpClient(client)
        parts = list(http_client.stream("/stream", "GET"))

    assert parts == [StreamPart(event="foo", data={"bar": 1})]

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/sdk-py/tests/test_errors.py
```py
from __future__ import annotations

from typing import cast

import httpx
import orjson
import pytest

from langgraph_sdk.errors import (
    APIStatusError,
    AuthenticationError,
    BadRequestError,
    ConflictError,
    InternalServerError,
    NotFoundError,
    PermissionDeniedError,
    RateLimitError,
    UnprocessableEntityError,
    _raise_for_status_typed,
)


def make_response(
    status: int,
    *,
    json_body: dict | None = None,
    text_body: str | None = None,
    headers: dict[str, str] | None = None,
) -> httpx.Response:
    request = httpx.Request("GET", "https://example.com/test")
    content: bytes | None
    if json_body is not None:
        content = orjson.dumps(json_body)
    elif text_body is not None:
        content = text_body.encode()
    else:
        content = b""
    return httpx.Response(
        status, headers=headers or {}, content=content, request=request
    )


@pytest.mark.parametrize(
    "status,exc_type",
    [
        (400, BadRequestError),
        (401, AuthenticationError),
        (403, PermissionDeniedError),
        (404, NotFoundError),
        (409, ConflictError),
        (422, UnprocessableEntityError),
        (429, RateLimitError),
        (500, InternalServerError),
        (503, InternalServerError),  # any 5xx
        (418, APIStatusError),  # unmapped 4xx falls back to base type
    ],
)
def test_raise_for_status_typed_maps_exceptions_and_sets_status_code(
    status: int, exc_type: type[APIStatusError]
) -> None:
    r = make_response(
        status, json_body={"message": "boom", "code": "abc", "param": "p", "type": "t"}
    )

    with pytest.raises(exc_type) as ei:
        _raise_for_status_typed(r)

    err = cast(APIStatusError, ei.value)
    assert err.status_code == status
    # response attribute should be present and match
    assert err.response.status_code == status


def test_request_id_is_extracted_when_present() -> None:
    r = make_response(
        404, json_body={"detail": "missing"}, headers={"x-request-id": "req-123"}
    )
    with pytest.raises(NotFoundError) as ei:
        _raise_for_status_typed(r)
    err = cast(APIStatusError, ei.value)
    # request_id only exists on APIStatusError subclasses
    assert err.request_id == "req-123"


def test_non_json_body_does_not_break_mapping() -> None:
    r = make_response(429, text_body="Too many requests")
    with pytest.raises(RateLimitError) as ei:
        _raise_for_status_typed(r)
    err = cast(APIStatusError, ei.value)
    assert err.status_code == 429


def test_field_extraction_from_json_body() -> None:
    r = make_response(
        400,
        json_body={
            "message": "Invalid parameter",
            "code": "invalid_param",
            "param": "limit",
            "type": "invalid_request_error",
        },
    )
    with pytest.raises(BadRequestError) as ei:
        _raise_for_status_typed(r)
    err = cast(APIStatusError, ei.value)
    assert err.code == "invalid_param"
    assert err.param == "limit"
    assert err.type == "invalid_request_error"

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/sdk-py/tests/test_select_fields_sync.py
```py
import functools
import json
import os
from pathlib import Path
from typing import get_args

from langgraph_sdk.schema import (
    AssistantSelectField,
    CronSelectField,
    RunSelectField,
    ThreadSelectField,
)

current_dir = os.path.dirname(os.path.abspath(__file__))


@functools.cache
def _load_spec() -> dict:
    with (
        Path(current_dir).parents[2]
        / "docs"
        / "docs"
        / "cloud"
        / "reference"
        / "api"
        / "openapi.json"
    ).open() as f:
        return json.load(f)


def _enum_from_request_select(spec: dict, path: str, method: str) -> set[str]:
    schema = spec["paths"][path][method]["requestBody"]["content"]["application/json"][
        "schema"
    ]
    if "properties" in schema:
        props = schema["properties"]
    elif "$ref" in schema:
        component = spec
        index = schema["$ref"].split("/")[1:]
        for part in index:
            component = component[part]
        props = component["properties"]
    else:
        raise ValueError(f"Unknown schema: {schema}")
    sel = props["select"]
    return set(sel["items"]["enum"])


def _enum_from_query_select(spec: dict, path: str, method: str) -> set[str]:
    params = spec["paths"][path][method]["parameters"]
    sel = next(p for p in params if p["name"] == "select")
    return set(sel["schema"]["items"]["enum"])


def test_assistants_select_enum_matches_sdk():
    spec = _load_spec()
    expected = set(get_args(AssistantSelectField))
    assert _enum_from_request_select(spec, "/assistants/search", "post") == expected


def test_threads_select_enum_matches_sdk():
    spec = _load_spec()
    expected = set(get_args(ThreadSelectField))
    assert _enum_from_request_select(spec, "/threads/search", "post") == expected


def test_runs_select_enum_matches_sdk():
    spec = _load_spec()
    expected = set(get_args(RunSelectField))
    assert _enum_from_query_select(spec, "/threads/{thread_id}/runs", "get") == expected


def test_crons_select_enum_matches_sdk():
    spec = _load_spec()
    expected = set(get_args(CronSelectField))
    assert _enum_from_request_select(spec, "/runs/crons/search", "post") == expected

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/sdk-py/tests/fixtures/response.txt
```txt
event: metadata
data: {"run_id":"01994ed8-29c4-7108-93d4-d649c7a9e370","attempt":1}

event: debug
data: {"step":-1,"timestamp":"2025-09-15T19:26:53.454492+00:00","type":"checkpoint","payload":{"config":{"configurable":{"checkpoint_ns":"","k":6,"host":"chat-langchain-v3-823d63bedfd35b5b8540dfc065f1c973.us.langgraph.app","accept":"*/*","origin":"https://smith.langchain.com","run_id":"01994ed8-29c4-7108-93d4-d649c7a9e370","referer":"https://smith.langchain.com/","user_id":"b9825b9a-aaad-4896-9b8d-c289030d3b03","graph_id":"chat","priority":"u=1, i","x-scheme":"https","sec-ch-ua":"\"Chromium\";v=\"140\", \"Not=A?Brand\";v=\"24\", \"Google Chrome\";v=\"140\"","thread_id":"23c8ebb7-3500-4f73-9b02-ab2184201bdf","x-real-ip":"10.0.0.161","x-user-id":"b9825b9a-aaad-4896-9b8d-c289030d3b03","user-agent":"Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/140.0.0.0 Safari/537.36","query_model":"anthropic/claude-3-5-haiku-20241022","x-tenant-id":"ebbaf2eb-769b-4505-aca2-d11de10372a4","assistant_id":"eb6db400-e3c8-5d06-a834-015cb89efe69","content-type":"application/json","x-request-id":"d9ae35cd43d17c9659a1189f43158167","authorization":"Bearer eyJhbGciOiJIUzI1NiIsImtpZCI6IkMyYWJyeEw1YVk2S3V6WHIiLCJ0eXAiOiJKV1QifQ.eyJpc3MiOiJodHRwczovL3V3c3hydHF1aWZnemFqdGJ3cGlrLnN1cGFiYXNlLmNvL2F1dGgvdjEiLCJzdWIiOiJiOTgyNWI5YS1hYWFkLTQ4OTYtOWI4ZC1jMjg5MDMwZDNiMDMiLCJhdWQiOiJhdXRoZW50aWNhdGVkIiwiZXhwIjoxNzU3OTY0NTQ1LCJpYXQiOjE3NTc5NjQyNDUsImVtYWlsIjoibnVub0BsYW5nY2hhaW4uZGV2IiwicGhvbmUiOiIiLCJhcHBfbWV0YWRhdGEiOnsicHJvdmlkZXIiOiJlbWFpbCIsInByb3ZpZGVycyI6WyJlbWFpbCIsImdvb2dsZSJdfSwidXNlcl9tZXRhZGF0YSI6eyJhdmF0YXJfdXJsIjoiaHR0cHM6Ly9saDMuZ29vZ2xldXNlcmNvbnRlbnQuY29tL2EvQUNnOG9jSlNaNnVkX0szRUdta3l5UWpxd2NxTlA3U0JOOURwaWNKdGlCX0JURHh4UC1haUZ3PXM5Ni1jIiwiY3VzdG9tX2NsYWltcyI6eyJoZCI6ImxhbmdjaGFpbi5kZXYifSwiZW1haWwiOiJudW5vQGxhbmdjaGFpbi5kZXYiLCJlbWFpbF92ZXJpZmllZCI6dHJ1ZSwiZnVsbF9uYW1lIjoiTnVubyBDYW1wb3MiLCJpc3MiOiJodHRwczovL2FjY291bnRzLmdvb2dsZS5jb20iLCJuYW1lIjoiTnVubyBDYW1wb3MiLCJwaG9uZV92ZXJpZmllZCI6ZmFsc2UsInBpY3R1cmUiOiJodHRwczovL2xoMy5nb29nbGV1c2VyY29udGVudC5jb20vYS9BQ2c4b2NKU1o2dWRfSzNFR21reXlRanF3Y3FOUDdTQk45RHBpY0p0aUJfQlREeHhQLWFpRnc9czk2LWMiLCJwcm92aWRlcl9pZCI6IjEwODAxMzI4NjAzNDU2MDY4NDcxMiIsInN1YiI6IjEwODAxMzI4NjAzNDU2MDY4NDcxMiJ9LCJyb2xlIjoiYXV0aGVudGljYXRlZCIsImFhbCI6ImFhbDEiLCJhbXIiOlt7Im1ldGhvZCI6Im9hdXRoIiwidGltZXN0YW1wIjoxNzU3OTYzNzA0fV0sInNlc3Npb25faWQiOiJmNzkyZWViZC1iZmFlLTQ1NjMtOTc0YS0wNWJjOWQ5ZjU3MjgiLCJpc19hbm9ueW1vdXMiOmZhbHNlfQ.AVbI9jO5Arbzzm5JUldGN6MXL6awWefxejLx99d9x9Y","x-auth-scheme":"langsmith","content-length":"5712","response_model":"anthropic/claude-3-5-haiku-20241022","sec-fetch-dest":"empty","sec-fetch-mode":"cors","sec-fetch-site":"cross-site","accept-encoding":"gzip, deflate, br, zstd","accept-language":"en-US,en;q=0.9","embedding_model":"openai/text-embedding-3-small","x-forwarded-for":"10.0.0.161","sec-ch-ua-mobile":"?0","x-forwarded-host":"chat-langchain-v3-823d63bedfd35b5b8540dfc065f1c973.us.langgraph.app","x-forwarded-port":"443","__after_seconds__":0,"x-forwarded-proto":"https","retriever_provider":"weaviate","sec-ch-ua-platform":"\"macOS\"","x-forwarded-scheme":"https","langgraph_auth_user":{"identity":"b9825b9a-aaad-4896-9b8d-c289030d3b03","is_authenticated":true,"display_name":"b9825b9a-aaad-4896-9b8d-c289030d3b03","kind":"StudioUser","permissions":["authenticated"]},"langgraph_request_id":"d9ae35cd43d17c9659a1189f43158167","router_system_prompt":"You are a LangChain Developer advocate. Your job is to help people using LangChain answer any issues they are running into.\n\nA user will come to you with an inquiry. Your first job is to classify what type of inquiry it is. The types of inquiries you should classify it as are:\n\n## `more-info`\nClassify a user inquiry as this if you need more information before you will be able to help them. Examples include:\n- The user complains about an error but doesn't provide the error\n- The user says something isn't working but doesn't explain why/how it's not working\n\n## `langchain`\nClassify a user inquiry as this if it can be answered by looking up information related to LangChain open source package. The LangChain open source package is a python library for working with LLMs. It integrates with various LLMs, databases and APIs.\n\n## `general`\nClassify a user inquiry as this if it is just a general question","general_system_prompt":"You are a LangChain Developer advocate. Your job is to help people using LangChain answer any issues they are running into.\n\nYour boss has determined that the user is asking a general question, not one related to LangChain. This was their logic:\n\n<logic>\n{logic}\n</logic>\n\nRespond to the user. Politely decline to answer and tell them you can only answer questions about LangChain-related topics, and that if their question is about LangChain they should clarify how it is.\n\nBe nice to them though - they are still a user!","langgraph_auth_user_id":"b9825b9a-aaad-4896-9b8d-c289030d3b03","response_system_prompt":"You are an expert programmer and problem-solver, tasked with answering any question about LangChain.\n\nGenerate a comprehensive and informative answer for the given question based solely on the provided search results (URL and content). Do NOT ramble, and adjust your response length based on the question. If they ask a question that can be answered in one sentence, do that. If 5 paragraphs of detail is needed, do that. You must only use information from the provided search results. Use an unbiased and journalistic tone. Combine search results together into a coherent answer. Do not repeat text. Cite search results using [${{number}}] notation. Only cite the most relevant results that answer the question accurately. Place these citations at the end of the individual sentence or paragraph that reference them. Do not put them all at the end, but rather sprinkle them throughout. If different results refer to different entities within the same name, write separate answers for each entity. For any citations, MAKE SURE to hyperlink them like [citation](source url) to make it easy for the user to click into the full docs.\n\n\n\nYou should use bullet points in your answer for readability. Put citations where they apply rather than putting them all at the end. DO NOT PUT THEM ALL THAT END, PUT THEM IN THE BULLET POINTS. REMEMBER: you should hyperlink any relevant source urls in the citations so that users can easily click into the full docs.\n\nIf there is nothing in the context relevant to the question at hand, do NOT make up an answer. Rather, tell them why you're unsure and ask for any additional information that may help you answer better.\n\nSometimes, what a user is asking may NOT be possible. Do NOT tell them that things are possible if you don't see evidence for it in the context below. If you don't see based in the information below that something is possible, do NOT say that it is - instead say that you're not sure.\n\nAnything between the following `context` html blocks is retrieved from a knowledge bank, not part of the conversation with the user.\n\n<context>\n    {context}\n<context/>","more_info_system_prompt":"You are a LangChain Developer advocate. Your job is help people using LangChain answer any issues they are running into.\n\nYour boss has determined that more information is needed before doing any research on behalf of the user. This was their logic:\n\n<logic>\n{logic}\n</logic>\n\nRespond to the user and try to get any more relevant information. Do not overwhelm them! Be nice, and only ask them a single follow up question.","__request_start_time_ms__":1757964413380,"langgraph_auth_permissions":["authenticated"],"research_plan_system_prompt":"You are a LangChain expert and a world-class researcher, here to assist with any and all questions or issues with LangChain, LangGraph, LangSmith, or any related functionality. Users may come to you with questions or issues.\n\nBased on the conversation below, generate a plan for how you will research the answer to their question.\n\nThe plan should generally not be more than 3 steps long, it can be as short as one. The length of the plan depends on the question.\n\nYou have access to the following documentation sources:\n- Conceptual docs\n- Integration docs\n- How-to guides\n\nYou do not need to specify where you want to research for all steps of the plan, but it's sometimes helpful.","generate_queries_system_prompt":"Generate 3 search queries to search for to answer the user's question.\n\nThese search queries should be diverse in nature - do not generate repetitive ones.","checkpoint_id":"1f09269e-f6c0-69e6-bfff-d88e995570cc"}},"parent_config":null,"values":{"messages":[],"documents":[]},"metadata":{"source":"input","step":-1,"parents":{}},"next":["__start__"],"tasks":[{"id":"18cb1707-4751-baec-5607-508b152c99c9","name":"__start__","interrupts":[],"state":null}],"checkpoint":{"checkpoint_id":"1f09269e-f6c0-69e6-bfff-d88e995570cc","thread_id":"23c8ebb7-3500-4f73-9b02-ab2184201bdf","checkpoint_ns":""},"parent_checkpoint":null}}

event: debug
