

AutowiredAnnotationBeanPostProcessor 实现 autowired 关键代码


org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#doCreateBean
		// Allow post-processors to modify the merged bean definition.
		synchronized (mbd.postProcessingLock) {
			if (!mbd.postProcessed) {
				applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
				mbd.postProcessed = true;
			}
		}

org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#applyMergedBeanDefinitionPostProcessors
org.springframework.beans.factory.annotation.AutowiredAnnotationBeanPostProcessor#postProcessMergedBeanDefinition


存放了 RootBeanDefinition 依赖的 bean，类型为 Set<Member> 其实是 Set<Field>
org.springframework.beans.factory.support.RootBeanDefinition#externallyManagedConfigMembers  
